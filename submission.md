# Mixtape Bug Hunt — Submission

## AI Usage

I used Claude throughout this project, primarily for codebase navigation, hypothesis testing, and verification — not for writing the fixes themselves, which are one- or two-line changes I made deliberately based on evidence.

**Codebase orientation (Milestone 1):** Before touching any bug, I had Claude read every file in `services/`, `routes/`, `models.py`, and `app.py`, and asked it to summarize what each module was responsible for and trace two real call chains end-to-end (a song rating → notification, and a playlist-songs lookup). This is what surfaced two suspicions before I ever reproduced anything: the asymmetry between `add_to_playlist()` and `rate_song()` (Issue #4), and the `songs[:-1]` slice in `get_playlist_songs()` (Issue #5). Both suspicions turned out to be correct, but I still reproduced each one live before fixing it, rather than trusting the read-through alone.

**Hypothesis testing during debugging (Milestone 2–3):** For each bug, Claude proposed a specific, falsifiable hypothesis and a `flask shell` script to test it directly against the real seeded database, rather than guessing at a fix. Two of these hypotheses were wrong, and catching that mattered:

- **Issue #2:** Claude's first theory was that SQLite drops timezone info on round-trip, causing the recency filter's comparison to misbehave. We tested it directly (checked whether a 34-hour-old event was correctly excluded) — it was, disproving the theory outright. Claude then re-read `seed_data.py`'s own inline comments and found the actual issue: the threshold value itself (24 hours) was simply too generous for a "listening now" feature, not a comparison bug.
- **Issue #2, second correction:** After the fix, one verification pass showed only 1 of 3 expected friends in a feed. Claude initially explained this away as "events aging out of the 30-minute window during the time we spent talking." **I pushed back and pointed out that only ~30 seconds had actually elapsed between seeding and checking — nowhere near enough time for that explanation to hold.** That correction was right, and it redirected the investigation toward re-running the check in a single uninterrupted session, which then confirmed the fix worked correctly. Claude's first explanation was a plausible-sounding guess, not a verified one — I did not accept it without checking the timing math myself.
- **Issue #3:** Claude's first hypothesis (a multi-tag `outerjoin` without `.distinct()` would return duplicate rows) was tested directly and returned `count: 1`, not the expected `3`. Rather than declaring victory or abandoning the issue, Claude ran the identical query via raw SQL execution (bypassing ORM entity hydration) and found the underlying query genuinely does produce 3 duplicate rows — SQLAlchemy's `Query.all()` was silently collapsing them by primary key in that particular session. This was later reconciled by running the project's own `pytest` suite (see Regression Tests below), where the same bug **did** reproduce as `3` results in a fresh in-memory database — showing the symptom's visibility was dependent on session/database state, not that the underlying defect wasn't real.

**Where I verified independently rather than trusting Claude's output:** For every fix, I ran the actual `flask shell` reproduction and verification scripts myself rather than accepting Claude's reasoning on its own — including catching a case where a fresh shell restart was needed because Python's module caching was silently serving a stale, pre-fix version of a file during re-testing (Issue #1). I also directly compared "before" and "after" notification counts myself for Issue #4 rather than accepting a description of expected behavior.

---

## Codebase Map

### Main files and their roles

- **`app.py`** — Flask application factory (`create_app`). Initializes the SQLAlchemy `db` instance, configures the SQLite database (`sqlite:///mixtape.db` by default), registers all four blueprints (`songs`, `playlists`, `users`, `feed`), and calls `db.create_all()` on startup.
- **`models.py`** — Defines all SQLAlchemy models: `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, plus three association tables: `friendships` (symmetric many-to-many), `song_tags` (many-to-many), and `playlist_entries` (many-to-many **with an explicit `position` column** — songs in a playlist have an ordered slot, not just insertion order). Every `DateTime` column defaults to `datetime.now(timezone.utc)` (timezone-aware at write time).
- **`routes/songs.py`** — Song search (`GET /songs/search`), song detail (`GET /songs/<id>`), rating (`POST /songs/<id>/rate`), and listening (`POST /songs/<id>/listen`). Delegates to `search_service`, `notification_service.rate_song`, and `streak_service.record_listening_event`.
- **`routes/playlists.py`** — Playlist creation, detail, song listing, and adding a song to a playlist. Delegates to `playlist_service` and `notification_service.add_to_playlist`.
- **`routes/users.py`** — User detail, streak lookup, notification listing, and marking a notification read. Delegates to `streak_service.get_streak` and `notification_service`.
- **`routes/feed.py`** — "Friends listening now" and general activity feed. Delegates entirely to `feed_service`.
- **`services/streak_service.py`** — Owns listening-streak logic: `record_listening_event` (logs a `ListeningEvent`, then calls `update_listening_streak`), `update_listening_streak` (increments/resets the streak based on days since last listen), `get_streak`.
- **`services/feed_service.py`** — `get_friends_listening_now` (friends who listened within a recency window, one most-recent song per friend) and `get_activity_feed` (last N friend events, no recency filter).
- **`services/search_service.py`** — `search_songs` (case-insensitive title/artist match, outer-joined to tags) and `get_song`.
- **`services/notification_service.py`** — `create_notification` (low-level insert), `add_to_playlist` (adds song to playlist + notifies original sharer), `rate_song` (upserts a rating), `get_notifications`, `mark_as_read`.
- **`services/playlist_service.py`** — `create_playlist`, `get_playlist_songs` (songs ordered by `position`), `get_playlist`, `get_user_playlists`.
- **`seed_data.py`** — Populates the DB with test users/songs/playlists for local development.
- **`tests/`** — Pytest coverage for playlists, search, and streaks (`test_playlists.py`, `test_search.py`, `test_streaks.py`).

### Pattern noticed

Every route file does **only** two things: parse the incoming request (JSON body / query params, presence validation) and format the response (`jsonify`, catching `ValueError` to return 404/400). There is **zero business logic in `routes/`** — all of it lives in `services/`. This matches the README's explicit note that "the bugs live in the `services/` layer": if something breaks at the HTTP/API level, the cause is always one layer down in the matching service file, never the route itself.

### Data flow — Example 1: a user rates a song

`POST /songs/<song_id>/rate` (`routes/songs.py`) parses `user_id` and `score` from the JSON body → calls `notification_service.rate_song(user_id, song_id, score)` → which validates `1 <= score <= 5`, looks up the `Song` and `User`, checks for an existing `Rating` (upsert via the `unique_user_song_rating` constraint on `user_id`+`song_id`), commits, and returns the `Rating`. The route serializes it back with `.to_dict()`.

`add_to_playlist()` (same file) follows a structurally similar pattern — a friend interacting with a song someone else shared — and ends with a call to `create_notification()` to alert the original sharer.

### Data flow — Example 2: a user views a playlist's songs

`GET /playlists/<playlist_id>/songs` (`routes/playlists.py`) → `playlist_service.get_playlist_songs(playlist_id)` → joins `Song` to the `playlist_entries` association table on `playlist_id`, orders ascending by the `position` column, and returns the list of songs.

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

**Reproduction steps:** In `flask shell`, called `update_listening_streak()` directly (isolated from the DB) with a fake user object whose `last_listened_at` was set to the Saturday immediately before the next upcoming Sunday, then passed `now` as that Sunday at noon UTC — simulating a user listening on two genuinely consecutive calendar days where the second day is a Sunday. Starting streak was `5`. Result: streak went to `1` instead of `6`.

**Navigation strategy:** Read `services/streak_service.py` during codebase orientation, before any reproduction. The increment branch read `elif days_since_last == 1 and today.weekday() != 6:` — the `and today.weekday() != 6` clause stood out immediately because the function's own docstring states the rule as simply "if the user listened yesterday: streak increments by 1," with no day-of-week exception mentioned anywhere. The moment of confidence came from the reproduction itself: a controlled Saturday→Sunday input produced exactly the failure the suspicious line predicted (`5 → 1` instead of `5 → 6`), confirming this was the actual cause, not just a suspicious area.

**Root cause:** Python's `datetime.weekday()` returns `6` for Sunday. The condition `days_since_last == 1 and today.weekday() != 6` correctly detects a consecutive-day listen, but then additionally requires that the *current* day not be a Sunday before incrementing. Every time a user's consecutive listening streak crossed into a Sunday, the code fell through to the `else` branch and reset the streak to `1` — even though the user had listened on genuinely back-to-back days. The correct behavior (per the function's own docstring) requires incrementing on *any* consecutive day, with no day-of-week exception; the extra condition contradicted the documented rule outright.

**Fix:** Removed the `and today.weekday() != 6` clause entirely, leaving `elif days_since_last == 1:`, restoring the exact rule stated in the docstring.

**Side-effect check:** Verified in a fresh `flask shell` session (a stale session initially gave a false negative, resolved by restarting so the edited file was actually re-imported) that Saturday→Sunday now correctly goes `5 → 6`. Also checked the adjacent "skip a day" boundary (`last_listened_at` set 3 days prior) — correctly resets to `1`, confirming that branch was untouched by the fix and still works. The "already listened today" no-op branch was not re-verified after the fix; noted as an outstanding check.

### Issue #2: Friends Listening Now shows people from yesterday

**Reproduction steps:** Used the real seed data (`python seed_data.py`), which contains deliberately-aged `ListeningEvent` rows with comments describing intended behavior ("within the past 30 minutes — should appear," "1–14 days ago — should NOT appear"). Called `get_friends_listening_now()` for `kenji`, whose friends are `nova` (only event: 2 hours old) and `aaliya` (only event: 34 hours old). Under the original `RECENT_THRESHOLD = timedelta(hours=24)`, `nova` appeared in the feed despite her event being 2 hours old.

**Navigation strategy:** First hypothesis — SQLite dropping timezone info on round-trip, causing inconsistent comparisons against the timezone-aware `cutoff` — was tested directly by checking whether `aaliya`'s 34-hour-old event was correctly excluded. It was, which disproved the comparison-bug theory. That redirected me to re-read `seed_data.py`'s own inline comments, which state that even 2-hour-old events "should NOT appear" — a claim that only makes sense if the intended "now" window is much narrower than 24 hours. Confirmed the intended value ("the past 30 minutes") directly, rather than guessing at a replacement number.

**Root cause:** `RECENT_THRESHOLD` was set to `timedelta(hours=24)`, but a "Friends Listening Now" live-presence feed is meant to reflect near-real-time activity. The query logic and filter mechanics were correct throughout — verified independently by running the exact filter by hand and confirming it matched the constant given to it. The bug was purely in the threshold *value*: 24 hours is wide enough that a friend's listening session from many hours earlier is presented as if it's happening right now, which is why users described stale activity as showing up as current.

**Fix:** Changed `RECENT_THRESHOLD` from `timedelta(hours=24)` to `timedelta(minutes=30)`.

**Side-effect check:** Verified both boundary directions in a single controlled session: `kenji`'s feed (2-hour and 34-hour-old friend events) correctly returns `[]` after the fix, and `nova`'s feed (three friends with 10–20-minute-old events) correctly returns all three. Also explicitly confirmed `get_activity_feed()` — the unfiltered activity feed — was not touched, since it intentionally has no recency filter by design per its own docstring, and changing a shared constant could plausibly have affected it if it had referenced `RECENT_THRESHOLD` too (it does not).

### Issue #3: The same song keeps showing up twice in search

**Reproduction steps:** `seed_data.py` deliberately seeds 5 songs with 3+ tags each. Called `search_songs("Crown Heights")` against a 3-tag song. This returned `count: 1` — not the `3` expected from a naive multi-tag-join hypothesis, an important negative result rather than a confirmation.

**Navigation strategy:** Tested and disproved two hypotheses before finding the real explanation. (1) Multi-tag `outerjoin` without `.distinct()` — disproved directly, as above. (2) To check whether the ORM was silently deduplicating, ran the identical query via raw SQL execution (`db.session.execute(q.statement)`, bypassing ORM entity hydration) and found the underlying SQL genuinely returns 3 duplicate rows for the same song ID — confirming the defect exists at the SQL level even though `Query(Model).all()` was collapsing it before it reached `.to_dict()`. (3) Also checked whether any song's title and artist both matched the same query term (the other possible "second code path" the assignment hints at) — none did, across all 13 seeded songs. The moment of confidence came from the raw-SQL execution step: it definitively separated "the join doesn't produce duplicates" from "the join produces duplicates but something is silently absorbing them."

**Root cause:** `search_songs()` joins `Song` to `song_tags` (a one-to-many relationship) without `.distinct()`, which is a textbook cause of duplicate rows on a one-to-many join — confirmed at the raw-SQL level (3 rows for a 3-tag song). Correct behavior requires the query to explicitly guard against this fan-out; relying on the ORM to absorb it silently is not a substitute, since that behavior is environment/version-dependent rather than guaranteed. Running the project's own `pytest` suite later reconciled why this was hard to see live: `test_search_no_duplicates_multi_tag_song` uses a fresh in-memory database per test rather than the shared file-based dev database used in manual `flask shell` testing, and that different session/database context does surface the duplicate as `3`, matching the test's own comment ("Should be 1, bug causes it to be 3").

**Fix:** Added `.distinct()` to the query in `search_songs()`, immediately after the `.filter(...)` clause.

**Side-effect check:** Verified a raw `SELECT DISTINCT` reference query for the same search condition returns `1` unique song ID, matching `search_songs("Crown Heights")` returning exactly `1` after the fix. Also ran a broad search (`search_songs("a")`) and confirmed it returns `13` results — matching the total number of seeded songs exactly, confirming `.distinct()` did not silently drop any legitimate, non-duplicate results elsewhere in the catalog.

### Issue #4: No notification when a friend rates my song

**Reproduction steps:** Found a song shared by `nova`, recorded her notification count, called `rate_song(darius.id, song.id, 4)` (darius rating nova's song), and re-checked the count. Count stayed identical — rating a friend's song created zero notifications for the sharer.

**Navigation strategy:** Found through direct code comparison, per the assignment's own hint that this root cause is architectural rather than a typo. Compared `rate_song()` against `add_to_playlist()` in the same file — both represent "a friend interacted with a song you shared," but only `add_to_playlist()` ends with a `create_notification()` call. Additional confirming evidence: the module's own top-of-file docstring already lists `'song_rated'` as an example `notification_type` ("e.g. 'song_added_to_playlist' or 'song_rated'"), meaning this notification type was planned but never wired into `rate_song()`.

**Root cause:** `rate_song()` performs the rating upsert and commits, then returns, with no call to `create_notification()` anywhere in the function under any condition. This is a missing step, not a broken conditional — the same file already has a correct, working template for it in `add_to_playlist()`, including a self-notification guard (`if song.shared_by != added_by_user_id`) that was never replicated for the rating path.

**Fix:** Added a `create_notification()` call at the end of `rate_song()`, mirroring `add_to_playlist()`'s exact pattern: a self-notification guard (`if song.shared_by != user_id`), notification type `"song_rated"`, and a body message in the same phrasing style as the existing notification type.

**Side-effect check:** Verified two cases in the same session: `darius` rating `nova`'s song correctly created a `song_rated` notification with the correct body text, and `nova` rating her own song correctly did **not** self-notify (count unchanged), confirming the guard condition behaves the same way it does in the existing, unmodified `add_to_playlist()` path.

### Issue #5: The last song in a playlist never shows up

**Reproduction steps:** Queried a real seeded playlist ("Late Night Vibes", 7 songs) two ways in the same session: once via the raw join/order-by directly against `playlist_entries` (ground truth — 7 songs, ending in "Free Throws"), and once via `get_playlist_songs()`. The function returned only 6 songs, missing "Free Throws" — the last song by `position`.

**Navigation strategy:** Spotted during codebase orientation, before any reproduction: `get_playlist_songs()` builds the songs query correctly, but the return statement was `[song.to_dict() for song in songs[:-1]]` — slicing off the last element of an already-correct, correctly-ordered list. The function's own docstring states "this function returns all songs in the playlist," directly contradicting what the code did — the moment of confidence was seeing that contradiction stated explicitly in the function's own documentation, not just inferring it from behavior.

**Root cause:** The query itself has no bug — it fetches and orders every song in the playlist correctly. The bug was purely in the final line, where `songs[:-1]` discarded the last item of the result list before converting to dicts, regardless of playlist size. Correct behavior (per the function's own stated contract) requires returning every song in the query result, not all-but-one.

**Fix:** Changed `songs[:-1]` to `songs`, removing the slice entirely.

**Side-effect check:** Verified two cases in a single fresh session: the 7-song playlist now returns all 7, ending correctly with "Free Throws"; and a **single-song playlist**, built specifically to stress-test the boundary this bug would hit hardest, now correctly returns `1` song instead of the empty list the old `[:-1]` logic would have produced (slicing "everything but the last" off a one-item list leaves nothing — a worse failure than the many-song case). Did not modify `get_playlist()` or `get_user_playlists()`, since neither touches the `songs[:-1]` line.

---

## Regression Tests

The starter repo already contains purpose-built regression tests for three of the five fixed issues, each with an inline comment documenting the expected pre-fix failure:

- **`tests/test_streaks.py::test_streak_increments_on_sunday`** — asserts that listening on a real Saturday/Sunday pair increments the streak to `2`. Under the original bug (Issue #1), this would have failed, since `today.weekday() != 6` would cause the code to reset the streak to `1` on Sunday instead.
- **`tests/test_search.py::test_search_no_duplicates_multi_tag_song`** — asserts a 3-tag song appears exactly once in search results, with the comment "Should be 1, bug causes it to be 3." Confirmed this test does reproduce the duplicate as `3` in its fresh in-memory database fixture (Issue #3), which is what motivated adding `.distinct()`.
- **`tests/test_playlists.py::test_playlist_returns_all_songs`** — asserts a 5-song playlist returns all 5, with the comment "Bug causes this to return 4." This directly exercises the `songs[:-1]` defect (Issue #5).

All three tests pass with the fixes applied (confirmed via `pytest tests/ -v`).

---

_(All 5 issues fixed, reproduced, and documented with complete root cause analysis entries. See git log on `bugfix/mixtape` for 5 separate `fix:` commits.)_
