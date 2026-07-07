# Mixtape Bug Hunt — Submission

## AI Usage

_(To be filled in after Milestone 4 — will describe specifically how Claude was used for navigation, tracing, and verification during this project, and where I overrode or double-checked its output.)_

---

## Codebase Map

### Main files and their roles

- **`app.py`** — Flask application factory (`create_app`). Initializes the SQLAlchemy `db` instance, configures the SQLite database (`sqlite:///mixtape.db` by default), registers all four blueprints (`songs`, `playlists`, `users`, `feed`), and calls `db.create_all()` on startup.
- **`models.py`** — Defines all SQLAlchemy models: `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, plus three association tables: `friendships` (symmetric many-to-many), `song_tags` (many-to-many), and `playlist_entries` (many-to-many **with an explicit `position` column** — songs in a playlist have an ordered slot, not just insertion order). Every `DateTime` column defaults to `datetime.now(timezone.utc)` — timezone-aware at write time, which matters because SQLite does not natively preserve timezone info on round-trip (flagged as a hypothesis for Issue #2 during orientation; later disproved by direct testing — see Issue #2 RCA below).
- **`routes/songs.py`** — Song search (`GET /songs/search`), song detail (`GET /songs/<id>`), rating (`POST /songs/<id>/rate`), and listening (`POST /songs/<id>/listen`). Delegates to `search_service` and `notification_service.rate_song` and `streak_service.record_listening_event`.
- **`routes/playlists.py`** — Playlist creation, detail, song listing, and adding a song to a playlist. Delegates to `playlist_service` and `notification_service.add_to_playlist`.
- **`routes/users.py`** — User detail, streak lookup, notification listing, and marking a notification read. Delegates to `streak_service.get_streak` and `notification_service`.
- **`routes/feed.py`** — "Friends listening now" and general activity feed. Delegates entirely to `feed_service`.
- **`services/streak_service.py`** — Owns listening-streak logic: `record_listening_event` (logs a `ListeningEvent`, then calls `update_listening_streak`), `update_listening_streak` (increments/resets the streak based on days since last listen), `get_streak`.
- **`services/feed_service.py`** — `get_friends_listening_now` (friends who listened within the recency window, one most-recent song per friend) and `get_activity_feed` (last N friend events, no recency filter).
- **`services/search_service.py`** — `search_songs` (case-insensitive title/artist match, outer-joined to tags) and `get_song`.
- **`services/notification_service.py`** — `create_notification` (low-level insert), `add_to_playlist` (adds song to playlist + notifies original sharer), `rate_song` (upserts a rating), `get_notifications`, `mark_as_read`.
- **`services/playlist_service.py`** — `create_playlist`, `get_playlist_songs` (songs ordered by `position`), `get_playlist`, `get_user_playlists`.
- **`seed_data.py`** — Populates the DB with test users/songs/playlists for local development. Several entries are deliberately constructed to expose specific issues (documented inline).
- **`tests/`** — Existing pytest coverage for playlists, search, and streaks (`test_playlists.py`, `test_search.py`, `test_streaks.py`).

### Pattern noticed

Every route file does **only** two things: parse the incoming request (JSON body / query params, presence validation) and format the response (`jsonify`, catching `ValueError` to return 404/400). There is **zero business logic in `routes/`** — all of it lives in `services/`. This means: if a bug reproduces at the HTTP/API level, the fix is never in the route; it's always one layer down in the matching service file. This matches the README's explicit note that "the bugs live in the `services/` layer."

### Data flow — Example 1: a user rates a song

`POST /songs/<song_id>/rate` (`routes/songs.py`) parses `user_id` and `score` from the JSON body → calls `notification_service.rate_song(user_id, song_id, score)` → which validates `1 <= score <= 5`, looks up the `Song` and `User`, checks for an existing `Rating` (upsert via the `unique_user_song_rating` constraint on `user_id`+`song_id`), commits, and returns the `Rating`. The route serializes it back with `.to_dict()`.

Notably, `rate_song()` originally had no call to `create_notification()` anywhere — compared line-by-line against `add_to_playlist()` (same file), which *does* end with a `create_notification(...)` call to notify the song's original sharer. Both functions represent "a friend interacted with your shared song," but only one of the two paths actually created a `Notification` row. This was the architectural gap behind Issue #4 (fixed — see RCA below).

### Data flow — Example 2: a user views a playlist's songs

`GET /playlists/<playlist_id>/songs` (`routes/playlists.py`) → `playlist_service.get_playlist_songs(playlist_id)` → joins `Song` to the `playlist_entries` association table on `playlist_id`, orders ascending by the `position` column, then returns the list. The original implementation sliced the result with `songs[:-1]` before converting to dicts — dropping whatever song was last in position order, regardless of playlist size. This was the root cause of Issue #5 (fixed — see RCA below).

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

**My fix and side-effect check:** Removed the `and today.weekday() != 6` clause entirely, leaving `elif days_since_last == 1:`. This restores the exact rule stated in the docstring, with no day-of-week exception. Verified in a fresh `flask shell` session (module caching had briefly given a false negative on the first re-test — resolved by restarting the shell so the edited file was actually re-imported): Saturday→Sunday consecutive listen now correctly goes `5 → 6`. Also checked the "skip a day" boundary (`last_listened_at` set 3 days prior) — correctly resets to `1`, confirming that branch was untouched by the fix. The "already listened today" (`days_since_last == 0`) no-op branch was not modified by this change; re-verification of that specific branch after the fix was not completed and is noted here as an outstanding check.

### Issue #2: Friends Listening Now shows people from yesterday

**How I reproduced it:** Used the real seed data (`python seed_data.py`) rather than fabricating test data, since `seed_data.py` already contains deliberately-aged `ListeningEvent` rows with comments describing intended behavior ("within the past 30 minutes — should appear," "1–14 days ago — should NOT appear"). Called `get_friends_listening_now()` directly in `flask shell` for `kenji`, whose friends are `nova` (only event: 2 hours old) and `aaliya` (only event: 34 hours old). Under the original `RECENT_THRESHOLD = timedelta(hours=24)`, `nova` appeared in the feed despite her event being 2 hours old — clearly not "now" by any reasonable definition, and old enough to plausibly read as "yesterday" to a user depending on time of day.

**How I found the root cause:** My first hypothesis (SQLite dropping timezone info on round-trip, causing `listened_at` comparisons against the timezone-aware `cutoff` to behave inconsistently) was directly tested and disproved: a controlled query correctly excluded `aaliya`'s 34-hour-old event even before any fix. That ruled out a comparison bug. Re-reading `seed_data.py`'s own inline comments made the actual root cause clear: events as recent as 2 hours old are explicitly commented as things that "should NOT appear" in the feed, which only makes sense if the intended definition of "listening now" is much narrower than the 24-hour value actually in the code. Confirmed with the user that the intended window is "the past 30 minutes," matching the seed data's "recent events" bucket exactly.

**The root cause:** `RECENT_THRESHOLD` was set to `timedelta(hours=24)`, but a "Friends Listening Now" / live-presence feed is meant to reflect near-real-time activity, not a full day of history. The query logic and filter mechanics were correct throughout — verified independently by running the exact filter by hand and confirming it matched the constant it was given. The bug was purely in the threshold *value*, not the comparison logic: 24 hours is wide enough that a friend's listening session from many hours earlier (which a user would describe as "yesterday") is incorrectly presented as if it's happening right now.

**My fix and side-effect check:** Changed `RECENT_THRESHOLD` from `timedelta(hours=24)` to `timedelta(minutes=30)`. Verified both boundary directions in a single controlled session: (1) `kenji`'s feed — whose only two friend-events are 2 hours and 34 hours old — correctly returns `[]` after the fix; (2) `nova`'s feed — whose three friends each have events 10–20 minutes old — correctly returns all three friends with their timestamps. Confirmed `get_activity_feed()` (the unfiltered activity feed) was not touched by this change, since it intentionally has no recency filter by design per its own docstring.

### Issue #3: The same song keeps showing up twice in search

**How I reproduced it:** `seed_data.py` deliberately seeds 5 songs with 3+ tags each, with an inline comment stating "these are the ones that expose Issue #3." I called `search_songs("Crown Heights")` directly in `flask shell`, targeting a song seeded with exactly 3 tags. This returned `count: 1`, not the `3` I expected from the tag-fanout hypothesis — an important negative result, not a confirmation.

**How I found the root cause (including two disproved hypotheses, kept here for honesty):** (1) First hypothesis: `search_songs()` does an `outerjoin` to the `song_tags` association table without `.distinct()`, which should fan out one row per tag on a one-to-many join. Tested directly: `search_songs("Crown Heights")` returned only 1 result, disproving that this manifests through the actual function call. (2) To check whether SQLAlchemy was silently deduplicating, I ran the identical query via `db.session.execute(q.statement)` (raw execution, bypassing ORM entity hydration) and got **3 duplicate rows** for the same song ID — confirming the underlying SQL genuinely does produce duplicates, but SQLAlchemy 2.0.51's `Query(Model).all()` was silently collapsing them by primary-key identity before they reached `.to_dict()`. (3) Second hypothesis: duplicates could come from a song matching *both* branches of the `or_(title, artist)` filter simultaneously. Checked all 13 seeded songs for title/artist word overlap — none existed. Also disproved. A broad search (`search_songs("a")`, matching most of the catalog) was checked for any repeated title across the board — none found.

**The root cause:** `search_songs()` joins `Song` to `song_tags` (a one-to-many relationship: one song can have multiple tag rows) but never adds `.distinct()` to collapse the resulting duplicate rows. This is a genuine, textbook defect — confirmed at the raw-SQL level to produce 3 rows for a 3-tag song. However, in this specific environment (SQLAlchemy 2.0.51, legacy `Query` API), calling `.all()` on an ORM entity query happens to deduplicate rows sharing the same primary key before returning them, which currently masks the defect from ever reaching the JSON response. **I could not get the literal user-facing symptom (duplicate JSON entries) to reproduce in this environment**, despite three separate, targeted attempts (direct multi-tag song, raw broad-catalog scan, and title/artist-overlap scan). The underlying code defect is real and verified via raw SQL; whether it's user-visible appears to depend on SQLAlchemy version/query-execution path (e.g., using `session.execute(select(...))` instead of the legacy `Query` API, or a different SQLAlchemy version, could plausibly bypass the entity-dedup safety net that's currently hiding it).

**My fix and side-effect check:** Added `.distinct()` to the query in `search_songs()`, immediately after the `.filter(...)` clause. This makes the query explicitly safe against duplicate rows from the one-to-many join, rather than depending on an implicit, version-specific ORM behavior that happens to hide the problem today but isn't guaranteed to in a different SQLAlchemy version or query style. Verified: (1) a raw `SELECT DISTINCT` reference query for the same search condition returns `1` unique song ID, matching (2) `search_songs("Crown Heights")` returning exactly `1` result after the fix, and (3) a broad search (`search_songs("a")`) returns `13` results — matching the total number of seeded songs exactly, confirming `.distinct()` did not drop any legitimate results.

### Issue #4: No notification when a friend rates my song

**How I reproduced it:** During codebase orientation, I compared `rate_song()` against `add_to_playlist()` in the same file and noticed `add_to_playlist()` ends with a `create_notification(...)` call while `rate_song()` has no equivalent. Confirmed with a live reproduction: found a song shared by `nova`, recorded `nova`'s notification count, called `rate_song(darius.id, song.id, 4)` (darius rating nova's song), and re-checked the count. Count stayed identical (`1 → 1` in the first pass) — rating a friend's song creates zero notifications for the sharer.

**How I found the root cause:** This was found through direct code comparison rather than execution tracing, per the assignment's own hint ("the root cause is architectural, not a typo... compare it line-by-line to the missing one"). `add_to_playlist()` and `rate_song()` are structurally parallel — both represent "a friend interacted with a song you shared" — but only one of them was ever wired up to call `create_notification()`. Additional confirming evidence: the module's own top-of-file docstring already lists `'song_rated'` as an example `notification_type` ("e.g. 'song_added_to_playlist' or 'song_rated'"), meaning this notification type was planned but the implementation was simply never added to `rate_song()`.

**The root cause:** `rate_song()` performs the rating upsert and commits, then returns — with no call to `create_notification()` anywhere in the function, under any condition. This isn't a broken conditional or an off-by-one; it's a step that was never implemented, even though the exact same file already has a correct, working template for it in `add_to_playlist()` (including the self-notification guard: `if song.shared_by != added_by_user_id`).

**My fix and side-effect check:** Added a `create_notification()` call at the end of `rate_song()`, after the existing `db.session.commit()`, mirroring `add_to_playlist()`'s exact pattern: guard against self-notification (`if song.shared_by != user_id`), notification type `"song_rated"` (matching the type already documented in the module docstring), and a body message following the same phrasing style ("{rater} rated your song '{title}' {score}/5."). Verified two cases in the same session: (1) `darius` rating `nova`'s song correctly created a `song_rated` notification for `nova`, with the correct body text; (2) `nova` rating her own song correctly did **not** create a self-notification (count unchanged, `2 → 2`), confirming the guard condition works the same way it does in the existing `add_to_playlist()` path. Did not modify `add_to_playlist()`, `create_notification()`, or any other function in the file.

### Issue #5: The last song in a playlist never shows up

**How I reproduced it:** Queried a real seeded playlist ("Late Night Vibes", 7 songs) two ways in the same `flask shell` session: once via the raw SQLAlchemy join/order-by directly against `playlist_entries` (the ground truth — 7 songs, ending in "Free Throws"), and once via the actual `get_playlist_songs()` function. The function returned only 6 songs, missing "Free Throws" — the last song by `position`.

**How I found the root cause:** Spotted during codebase orientation, before any reproduction: `get_playlist_songs()` builds the songs query correctly (joined to `playlist_entries`, ordered ascending by `position`), but the return statement was `[song.to_dict() for song in songs[:-1]]` — slicing off the last element of an already-correct, correctly-ordered list. The function's own docstring includes the line "Note: This function returns all songs in the playlist," which directly contradicts what the code did, confirming this was an unintentional defect rather than a documented design choice.

**The root cause:** The query itself has no bug — it fetches and orders every song in the playlist correctly. The bug was purely in the final line, where `songs[:-1]` discarded the last item of the result list before converting to dicts, regardless of playlist size. This dropped the last song by `position` from every playlist, every time, independent of how many songs the playlist has.

**My fix and side-effect check:** Changed `songs[:-1]` to `songs`, removing the slice entirely so the full, correctly-ordered query result is returned as-is. Verified two cases in a single fresh session: (1) the 7-song playlist now returns all 7 songs, ending correctly with "Free Throws"; (2) a boundary case specifically built to stress-test this fix — a playlist with exactly **one** song — now correctly returns `1` song. This boundary case matters because under the old `[:-1]` logic, a single-song playlist would have returned an **empty list**, not just a list short by one — slicing "everything but the last" off a one-item list leaves nothing. Did not modify `get_playlist()` or `get_user_playlists()`, and did not need to — neither touches the `songs[:-1]` line.

---

_(All five issues fixed and documented. Regression tests and final AI usage writeup pending — Milestone 4.)_
