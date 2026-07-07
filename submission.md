# Mixtape Bug Hunt — Submission

## AI Usage

_(To be filled in after Milestones 2–4 — will describe specifically how Claude was used for navigation, tracing, and verification during this project, and where I overrode or double-checked its output.)_

---

## Codebase Map

### Main files and their roles

- **`app.py`** — Flask application factory (`create_app`). Initializes the SQLAlchemy `db` instance, configures the SQLite database (`sqlite:///mixtape.db` by default), registers all four blueprints (`songs`, `playlists`, `users`, `feed`), and calls `db.create_all()` on startup.
- **`models.py`** — Defines all SQLAlchemy models: `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, plus three association tables: `friendships` (symmetric many-to-many), `song_tags` (many-to-many), and `playlist_entries` (many-to-many **with an explicit `position` column** — songs in a playlist have an ordered slot, not just insertion order). Every `DateTime` column defaults to `datetime.now(timezone.utc)` — timezone-aware at write time, which matters because SQLite does not natively preserve timezone info on round-trip (flagged as a hypothesis for Issue #2, unverified as of this point).
- **`routes/songs.py`** — Song search (`GET /songs/search`), song detail (`GET /songs/<id>`), rating (`POST /songs/<id>/rate`), and listening (`POST /songs/<id>/listen`). Delegates to `search_service` and `notification_service.rate_song` and `streak_service.record_listening_event`.
- **`routes/playlists.py`** — Playlist creation, detail, song listing, and adding a song to a playlist. Delegates to `playlist_service` and `notification_service.add_to_playlist`.
- **`routes/users.py`** — User detail, streak lookup, notification listing, and marking a notification read. Delegates to `streak_service.get_streak` and `notification_service`.
- **`routes/feed.py`** — "Friends listening now" and general activity feed. Delegates entirely to `feed_service`.
- **`services/streak_service.py`** — Owns listening-streak logic: `record_listening_event` (logs a `ListeningEvent`, then calls `update_listening_streak`), `update_listening_streak` (increments/resets the streak based on days since last listen), `get_streak`.
- **`services/feed_service.py`** — `get_friends_listening_now` (friends who listened within the last 24h, one most-recent song per friend) and `get_activity_feed` (last N friend events, no recency filter).
- **`services/search_service.py`** — `search_songs` (case-insensitive title/artist match, outer-joined to tags) and `get_song`.
- **`services/notification_service.py`** — `create_notification` (low-level insert), `add_to_playlist` (adds song to playlist + notifies original sharer), `rate_song` (upserts a rating — **does not create a notification**), `get_notifications`, `mark_as_read`.
- **`services/playlist_service.py`** — `create_playlist`, `get_playlist_songs` (songs ordered by `position` — currently sliced with `[:-1]`, dropping the last song), `get_playlist`, `get_user_playlists`.
- **`seed_data.py`** — Populates the DB with test users/songs/playlists for local development.
- **`tests/`** — Existing pytest coverage for playlists, search, and streaks (`test_playlists.py`, `test_search.py`, `test_streaks.py`).

### Pattern noticed

Every route file does **only** two things: parse the incoming request (JSON body / query params, presence validation) and format the response (`jsonify`, catching `ValueError` to return 404/400). There is **zero business logic in `routes/`** — all of it lives in `services/`. This means: if a bug reproduces at the HTTP/API level, the fix is never in the route; it's always one layer down in the matching service file. This matches the README's explicit note that "the bugs live in the `services/` layer."

### Data flow — Example 1: a user rates a song

`POST /songs/<song_id>/rate` (`routes/songs.py`) parses `user_id` and `score` from the JSON body → calls `notification_service.rate_song(user_id, song_id, score)` → which validates `1 <= score <= 5`, looks up the `Song` and `User`, checks for an existing `Rating` (upsert via the `unique_user_song_rating` constraint on `user_id`+`song_id`), commits, and returns the `Rating`. The route serializes it back with `.to_dict()`.

Notably, `rate_song()` has no call to `create_notification()` anywhere — compared line-by-line against `add_to_playlist()` (same file), which *does* end with a `create_notification(...)` call to notify the song's original sharer. Both functions represent "a friend interacted with your shared song," but only one of the two paths actually creates a `Notification` row. This is the architectural gap behind Issue #4.

### Data flow — Example 2: a user views a playlist's songs

`GET /playlists/<playlist_id>/songs` (`routes/playlists.py`) → `playlist_service.get_playlist_songs(playlist_id)` → joins `Song` to the `playlist_entries` association table on `playlist_id`, orders ascending by the `position` column, then returns the list. The current implementation slices the result with `songs[:-1]` before converting to dicts — this drops whatever song is last in position order, regardless of playlist size. This is the suspected root cause of Issue #5, to be confirmed by reproduction before fixing.

### Five open issues (from `README.md`)

| # | Title | Affected service |
|---|-------|-------------------|
| 1 | Listening streak keeps resetting | `streak_service.py` |
| 2 | Friends Listening Now shows people from yesterday | `feed_service.py` |
| 3 | Same song shows up twice in search | `search_service.py` |
| 4 | No notification when a friend rates my song (works for playlist-add) | `notification_service.py` |
| 5 | Last song in a playlist never shows up | `playlist_service.py` |

---

## Root Cause Analysis Entries

### Issue #1: My listening streak keeps resetting

**How I reproduced it:** In `flask shell`, called `update_listening_streak()` directly (isolated from the DB) with a fake user object whose `last_listened_at` was set to the Saturday immediately before the next upcoming Sunday, then passed `now` as that Sunday at noon UTC. This simulates a user listening on two genuinely consecutive calendar days, where the second day is a Sunday. Starting streak was `5`.

**How I found the root cause:** Read `services/streak_service.py` during codebase orientation (before any reproduction). The increment branch read `elif days_since_last == 1 and today.weekday() != 6:` — the `and today.weekday() != 6` clause stood out immediately because the function's own docstring states the rule as simply "if the user listened yesterday: streak increments by 1," with no day-of-week exception mentioned anywhere. Confirmed this was the actual cause (not just a suspicious area) by reproducing with a controlled Saturday→Sunday input: the streak went `5 → 1` instead of `5 → 6`, exactly matching the pattern predicted by the code.

**The root cause:** Python's `datetime.weekday()` returns `6` for Sunday. The condition `days_since_last == 1 and today.weekday() != 6` correctly detects a consecutive-day listen, but then additionally requires that the *current* day not be a Sunday before incrementing. Because of this, every single time a user's consecutive listening streak crossed into a Sunday, the code fell through to the `else` branch and reset the streak to `1`, even though the user had listened on back-to-back days exactly as the streak is supposed to reward. This explains the user-reported symptom ("keeps resetting") — it wasn't intermittent, it was guaranteed to happen once a week for every active listener.

**My fix and side-effect check:** Removed the `and today.weekday() != 6` clause entirely, leaving `elif days_since_last == 1:`. This restores the exact rule stated in the docstring, with no day-of-week exception. Verified in a fresh `flask shell` session (module caching had briefly given a false negative on the first re-test — resolved by restarting the shell so the edited file was actually re-imported): Saturday→Sunday consecutive listen now correctly goes `5 → 6`. Also checked the "skip a day" boundary (`last_listened_at` set 3 days prior) — correctly resets to `1`, confirming that branch was untouched by the fix. The "already listened today" (`days_since_last == 0`) no-op branch was not modified by this change and was not re-verified after the fix — noting this as an outstanding check before final submission, since the assignment asks to verify both sides of a boundary condition.

_(Additional entries to be added for Issues #2–#5 as we work through them.)_
