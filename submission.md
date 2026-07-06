# Mixtape Bug Hunt — Submission

## AI Usage

- Used AI to get oriented on each service file before touching any issue —
  asked "what is this module responsible for and what does each function do"
  for each file in `services/`.
- Used AI to help draft the initial codebase map structure after I had
  already read the files myself.
- Used AI to help design a reproduction plan for Issue #1 (constructing a
  Saturday last_listened_at, then simulating a Sunday listen via
  update_listening_streak directly in a flask shell) since the bug only
  triggers on a specific day-of-week boundary that wouldn't occur naturally
  on the day I was testing.
- Verified the root cause myself by tracing through the elif/else branches
  by hand rather than taking AI's first explanation at face value — asked
  it to explain what `today.weekday() != 6` excludes, then worked through
  why that specific exclusion causes a fall-through to the reset branch on
  a legitimate one-day gap.
- While reproducing Issue #5, ran into an unrelated error when trying to
  build a fresh test playlist via `add_to_playlist` (a NOT NULL constraint
  failure on `playlist_entries.position`). Used AI to help read the
  SQLAlchemy traceback and identify that `playlist.songs.append(song)`
  only populates the two foreign key columns on the association table,
  not the extra `position`/`added_by` columns, which are NOT NULL with no
  default. Worked around it by testing against playlists already
  populated by `seed_data.py` instead of creating new ones through the
  broken path.

---

## Codebase Map

### Main files and their roles

`app.py` is the Flask application factory. It creates the app, configures
the SQLite database via `SQLALCHEMY_DATABASE_URI`, initializes the `db`
object used everywhere else, and registers four blueprints under fixed URL
prefixes: `/songs`, `/playlists`, `/users`, `/feed`. There is no route
registered at `/`, so hitting the root URL returns a 404 by design — this
isn't a bug, just an app with no homepage.

`models.py` defines six SQLAlchemy models: `User`, `Tag`, `Song`,
`ListeningEvent`, `Rating`, `Playlist`, and `Notification`, plus three
association tables. `friendships` is a symmetric self-referential
many-to-many on `User` (a user's `friends` relationship). `song_tags` links
`Song` to `Tag`. `playlist_entries` links `Playlist` to `Song` but also
carries extra columns — `position`, `added_by`, `added_at` — meaning
playlist order is an explicit stored value, not just insertion order.
Notably, `listening_streak` and `last_listened_at` live directly on the
`User` row rather than in a separate streak table, and `Rating` has a
database-level unique constraint on `(user_id, song_id)`, so a user can
never have two Rating rows for the same song.

`routes/` contains four blueprint files — `songs.py`, `playlists.py`,
`users.py`, `feed.py` — one per resource. Every route function is thin: it
parses the request, calls into the matching `services/` function, and
formats the JSON response. No business logic lives in the routes
themselves.

`services/` contains the actual logic, split as follows:
- `streak_service.py` — increments or resets `User.listening_streak` based
  on gaps between listening events.
- `feed_service.py` — builds the "listening now" feed (rolling 24-hour
  window, deduplicated to one song per friend) and the general "activity
  feed" (no time filter, no dedup, just latest N events).
- `search_service.py` — case-insensitive song search over title/artist,
  joined to tags.
- `notification_service.py` — the generic `create_notification` writer,
  plus two feature-specific functions (`add_to_playlist`, `rate_song`) that
  live here because they happen to trigger (or should trigger)
  notifications, even though the actions themselves belong conceptually to
  playlists and ratings.
- `playlist_service.py` — playlist creation and retrieval, including the
  ordered song list.

`seed_data.py` populates the database with test users, songs, ratings,
listening events, and friendships so the app has data to exercise on first
run.

### Data flow — rating a song

`POST /songs/<song_id>/rate` hits `routes/songs.py::rate`, which pulls
`user_id` and `score` from the request body, then calls
`notification_service.rate_song(user_id, song_id, score)`. That function
validates the score is 1–5, looks up the song and user, checks for an
existing `Rating` row for that `(user_id, song_id)` pair (upserting if
found, inserting if not — the DB's unique constraint backs this up),
commits, and returns the `Rating`. Notably, `rate_song` never calls
`create_notification`, even though the sibling function `add_to_playlist`
(triggered from `POST /playlists/<id>/songs`) follows almost the same shape
— mutate the data, then call `create_notification` to tell the song's
original sharer — but stops short of that last step.

### Patterns noticed

Routes are uniformly thin — all business logic is pushed into `services/`,
and every route wraps its service call in a `try/except ValueError` to
translate domain errors into 404/400 responses. Several services perform
database queries without deduplication or slicing safeguards despite
manipulating ordered or joined data (e.g., an `outerjoin` against a
many-to-many tag table with no `.distinct()`, or a query result sliced
with `[:-1]` despite the docstring claiming "all songs" are returned).
`notification_service.py` doesn't map 1:1 to a single feature — it
centralizes anything that triggers a notification, regardless of which
resource the action is really about.

### Additional issue found (not one of the five, not yet fixed)

While reproducing Issue #5, discovered that `add_to_playlist` in
`notification_service.py` cannot actually add new songs to a playlist.
`playlist.songs.append(song)` uses SQLAlchemy's simple many-to-many
`.append()`, which only populates the `playlist_id` and `song_id` columns
on the `playlist_entries` association table. That table also defines
`position` and `added_by` as NOT NULL with no default, so the resulting
INSERT fails with `sqlite3.IntegrityError: NOT NULL constraint failed:
playlist_entries.position`. Worked around this during testing by using
playlists pre-populated by `seed_data.py` (which presumably inserts
directly into `playlist_entries` with all columns specified) rather than
adding songs through the API. Not fixed as part of this submission since
it isn't one of the five tracked issues, but noted here since it's a real,
reproducible bug with clear user impact — nobody can add a song to a
playlist through the app right now.

---

## Root Cause Analysis

### Issue #1: My listening streak keeps resetting

**How I reproduced it:**
In a `flask shell`, set a test user's `last_listened_at` to the most recent
Saturday and `listening_streak` to 5. Called `update_listening_streak(user,
sunday)` directly, where `sunday` was the day immediately after that
Saturday — simulating the user listening again on a genuine one-day gap.
Expected the streak to increment to 6. Instead it reset to 1, confirming
the bug.

**How I found the root cause:**
Read `streak_service.py` during initial orientation and noticed the
condition `elif days_since_last == 1 and today.weekday() != 6`. Confirmed
via the reproduction above that a Saturday→Sunday one-day gap hits this
exact branch. Traced through the if/elif/else by hand: when
`days_since_last == 1` is true but `today.weekday() != 6` is false (i.e.
today is Sunday), the elif condition as a whole is false, so execution
falls through to the `else` branch.

**The root cause:**
`update_listening_streak` only increments the streak when
`days_since_last == 1 and today.weekday() != 6`. Since Python's
`datetime.weekday()` returns `6` for Sunday, this condition is false
whenever "today" is a Sunday — even when the gap since the last listen is
exactly one day, which should always increment per the function's own
docstring. When a user listens on a Sunday after listening on Saturday,
`days_since_last == 1` is true but `today.weekday() != 6` is false, so the
`elif` doesn't fire, and execution falls through to the `else` branch,
which resets `listening_streak` to `1` instead of incrementing it. Any
user who listens every day, including Sundays, has their streak wiped
weekly.

**My fix and side-effect check:**
Removed the `and today.weekday() != 6` condition from the `elif`, so it now
reads `elif days_since_last == 1:`. The streak now increments on any
one-day gap regardless of the day of the week, matching the function's
documented intent. Verified by re-running the Saturday→Sunday reproduction
(streak now correctly goes from 5 to 6 instead of resetting to 1), and
checked the other branches were unaffected: calling the function twice on
the same day is still a no-op, a 3-day gap still resets the streak to 1,
and a normal non-Sunday one-day gap (Tuesday→Wednesday) still increments
correctly.

---

### Issue #5: The last song in a playlist never shows up

**How I reproduced it:**
Found three seeded playlists with 7 songs each via a query joining
`Playlist` to `playlist_entries` and counting songs per playlist. Picked
one (`84ccd2aa-2f22-48c2-ab12-39eee9e3184c`) and, in a `flask shell`, ran
the same query `get_playlist_songs` uses internally — joining `Song` to
`playlist_entries`, filtering by playlist ID, ordering by `position`
ascending — directly, without going through the service function. That
returned all 7 songs in correct order, ending with "Harlem Renaissance."
Then called `get_playlist_songs(pid)` itself and got back only 6 songs,
missing "Harlem Renaissance" — the last song in position order.

**How I found the root cause:**
Read `playlist_service.py` during initial orientation and noticed the
return line `return [song.to_dict() for song in songs[:-1]]`, which
contradicted the function's own docstring ("This function returns all
songs in the playlist"). Confirmed by comparing the raw query result (7
songs) against the service function's output (6 songs) on the same
playlist ID — the two queries are otherwise identical, so the only place
a song could be dropped is the final list comprehension.

**The root cause:**
`get_playlist_songs` correctly queries and orders every song in the
playlist by `position`, but the return statement slices the resulting
list with `songs[:-1]` before converting to dicts, which drops the last
element of the list every time, regardless of playlist length. Since the
songs are ordered ascending by position, the dropped song is always the
one with the highest position value — i.e., the last song added to the
playlist. This affects every playlist with at least one song; a playlist
of length 1 would return an empty list, and a playlist of length 7 (as
tested) returns 6.

**My fix and side-effect check:**
Changed `songs[:-1]` to `songs` in the return statement, so the full
ordered list is converted to dicts and returned. Verified by re-running
the reproduction on the same seeded playlist: `get_playlist_songs` now
returns all 7 songs, ending with "Harlem Renaissance," matching the raw
query result exactly. Also checked the other two seeded playlists with 7
songs each and confirmed both now return all 7 songs in the correct order.
Did not find any other code that depends on the previous (buggy) shorter
length, so no related functionality appears to rely on the missing last
song.