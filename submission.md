# Mixtape Bug Hunt — Submission

## AI Usage

I used Claude throughout this project during codebase navigation and debugging, not for generating fixes wholesale. Specific ways it helped:

- **File summaries:** I pasted `models.py`, `notification_service.py`, `streak_service.py`, and `search_service.py` and asked what each module was responsible for, to build a mental model faster than reading cold.
- **Line-by-line comparisons:** For Issue #4 (missing rating notification), I asked Claude to compare `rate_song()` against `add_to_playlist()` in `notification_service.py` since the two functions handle similar "user does X to someone else's song" cases. It pointed out that `add_to_playlist()` ends with a `create_notification()` call and `rate_song()` doesn't — that's the whole bug, an architectural gap rather than a broken line of code.
- **Explaining query semantics:** For Issue #3 (duplicate search results), I didn't immediately see why `search_songs()` in `search_service.py` would return dupes. I asked Claude to explain what the `.outerjoin(song_tags, ...)` was doing in that query and why it wasn't paired with any `Tag`-related filtering or select. It explained that a plain SQL join fans out one result row per matching joined row — so songs with 2+ tags produce 2+ result rows — which is why only some songs (the multi-tagged ones) show duplicates and not all of them.
- **Explaining stdlib behavior:** For Issue #1 (streak resets), I asked Claude to explain what `datetime.weekday()` returns and confirmed `6 == Sunday`. That let me see that `today.weekday() != 6` in `update_listening_streak()` was silently blocking streak increments specifically on Sundays.

**Where I verified independently:** For all three issues, I read the actual function bodies myself before accepting the explanation, and confirmed the root cause against the DB schema in `models.py` (e.g., confirming `playlist_entries` has a composite key, confirming `Rating` has a `UniqueConstraint`) rather than taking the AI's read of the schema at face value. I still need to reproduce each bug by actually hitting the running app / DB before finalizing these entries (see note in each RCA below), and I need to implement, test, and commit each fix separately per the branch/commit requirements.

---

## Codebase Map

**`app.py`** — Flask app factory (`create_app`) and SQLAlchemy `db` setup. Everything else imports `db` from here.

**`models.py`** — Defines 6 SQLAlchemy models: `User`, `Tag`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, plus 3 association tables:

- `friendships` — symmetric many-to-many between users.
- `song_tags` — plain many-to-many between songs and tags.
- `playlist_entries` — many-to-many between playlists and songs, but with extra columns: `position` (explicit ordering), `added_by`, and `added_at`. The primary key is the composite `(playlist_id, song_id)`, so the same song can't appear twice in the same playlist at the DB level.

Notable schema details:

- `User.listening_streak` and `User.last_listened_at` are stored directly as columns on `User` — the streak is a running counter updated on each listen, not recomputed from `ListeningEvent` history.
- `Rating` has a `UniqueConstraint(user_id, song_id)` — one rating per user per song; rating again updates the existing row rather than creating a new one.
- `Notification` is generic (`notification_type` + `body` strings) rather than polymorphic per event — every service that wants to notify a user constructs a `Notification` manually via `create_notification()`.
- Every model has a `to_dict()` method; routes serialize by calling this rather than building responses by hand.

**`routes/`** — Thin layer: each route parses the request, calls exactly one service function, and returns JSON. No business logic lives here.

- `songs.py` — search, get song, rate song (`POST /<song_id>/rate`), record a listen (`POST /<song_id>/listen`).
- `playlists.py` — playlist creation and song management.
- `users.py` — user profiles, streaks, notifications.
- `feed.py` — friends listening now, activity feed.

**`services/`** — All business logic. This is where the 5 tracked bugs live.

- `streak_service.py` — `record_listening_event()` creates a `ListeningEvent` and calls `update_listening_streak()`, which compares `last_listened_at` to `now` to decide whether to increment, hold, or reset the streak.
- `search_service.py` — `search_songs()` queries `Song` by title/artist substring match.
- `notification_service.py` — `create_notification()` is the shared primitive; `add_to_playlist()` and `rate_song()` are supposed to call it after their respective actions, though (see Issue #4 below) only one of them actually does.
- `feed_service.py` — friends' recent listening activity (not yet reviewed in depth).
- `playlist_service.py` — playlist retrieval, presumably ordering songs by `position` from `playlist_entries` (not yet reviewed in depth).

### Data flow — user rates a song

`POST /songs/<song_id>/rate` in `routes/songs.py` parses `user_id` and `score` from the request body and calls `notification_service.rate_song(user_id, song_id, score)`. That function validates the score is 1–5, looks up the `Song` and `User`, checks for an existing `Rating` row (upsert via the unique constraint), and commits. **It does not call `create_notification()`** — unlike the structurally similar `add_to_playlist()`, which notifies the song's original sharer as its last step. This gap is Issue #4.

### Pattern noticed

Routes are uniformly thin — all input parsing and response formatting happens in `routes/`, all business logic happens in `services/`. Several services (`notification_service.py`, `streak_service.py`) follow a "fetch entities → validate → mutate → commit" shape, which makes it easy to spot when one function skips a step (like the notification call) that a structurally similar function performs.

---

## Root Cause Analysis Entries

### Issue #1: My listening streak keeps resetting

**How I reproduced it:** In a `flask shell` session, I grabbed a seeded user (`nova`), set `listening_streak` to 5 and `last_listened_at` to the most recent Saturday (computed relative to the next upcoming Sunday), then called `update_listening_streak(user, next_sunday)` directly with `next_sunday` being exactly one calendar day after `last_listened_at`. This is a legitimate consecutive-day listen, so the streak should have incremented to 6. Instead, `user.listening_streak` came back as `1` after the call — confirming the reset-on-Sunday bug.

**How I found the root cause:** Read `streak_service.py` top to bottom. `update_listening_streak()` computes `days_since_last = (today - last_date).days` and branches on it. The `days_since_last == 1` branch — the one that should always increment the streak for a consecutive day — has an extra condition: `and today.weekday() != 6`. That extra clause doesn't appear anywhere in the documented streak rules in the function's own docstring, which was the tell that it didn't belong.

**The root cause:** Python's `datetime.weekday()` returns `6` for Sunday. The increment branch is gated by `days_since_last == 1 and today.weekday() != 6`, meaning that even when a user listens on consecutive calendar days, the streak will **not** increment if the second day happens to be a Sunday — it falls through to the `else` clause instead and resets to 1. This directly contradicts the stated rule ("increments by 1" on a consecutive day) and has no legitimate reason to exist based on the function's documentation.

**My fix and side-effect check:** Removed the `and today.weekday() != 6` clause so the condition is simply `elif days_since_last == 1:`. Verified directly: (1) re-ran the Sunday repro after the fix — the streak now correctly increments from 5 to 6 instead of resetting to 1; (2) confirmed a 3-day gap still correctly resets the streak to 1, so the skip-a-day reset logic is untouched; (3) the `days_since_last == 0` "already listened today" branch was not touched by this change.

---

### Issue #2: Friends Listening Now shows people from yesterday

**Note on process:** I originally planned to fix Issue #3 (duplicate search results) as my third bug. I traced the code to `search_service.py`'s `.outerjoin(song_tags, ...)` in `search_songs()`, hypothesizing that the join fans out one row per tag and causes duplicate `Song` entities in results for multi-tagged songs. I tested this directly — both in an isolated repro script and against the real seeded multi-tag songs (e.g. "Crown Heights Anthem," 3 tags) via `curl` — and confirmed via SQLAlchemy's own documentation that `db.session.query(Song)` (a full-entity legacy `Query`) automatically deduplicates results by primary key regardless of join fan-out. The join is dead code, but it is **not** the cause of Issue #3 in this codebase/library version. Per the assignment's guidance ("if you can't reproduce a bug after a genuine attempt, try a different one"), I switched to Issue #2 as my third bug instead.

**How I reproduced it:** Read `feed_service.py` and noticed `RECENT_THRESHOLD = timedelta(hours=24)`. Cross-referencing `seed_data.py`'s comment ("Older events (1–14 days ago)... should NOT appear in listening now after fix") against the actual seeded timestamps (2h, 10h, 18h, 26h, 34h, 42h, 50h, 58h old), several of those ages fall _inside_ a 24-hour window, yet the seed data's own comment says none should appear. Initially, calling `get_friends_listening_now()` for `nova` only showed very recent (10–30 min old) events, because the function's dedup logic keeps only the most-recent event per friend, and each friend's freshest event happened to be recent — masking the bug. To force the actual condition, I deleted user `kenji`'s other listening events and gave him a single event timestamped 20 hours ago (comfortably "yesterday," but still under the 24-hour cutoff). Calling `get_friends_listening_now(nova.id)` again returned `kenji`, `Midnight Drive`, `2026-07-06T05:26:55` — a 20-hour-old event surfaced in a feed meant to represent who's listening _right now_.

**How I found the root cause:** Read `feed_service.py`'s `get_friends_listening_now()`. It queries `ListeningEvent` for all friends where `listened_at >= cutoff`, with `cutoff = now - RECENT_THRESHOLD`. The query logic and dedup-by-most-recent-event are both correct in isolation — there's no off-by-one or comparison bug. The problem is the value of `RECENT_THRESHOLD` itself: 24 hours.

**The root cause:** `RECENT_THRESHOLD` is set to `timedelta(hours=24)`, meaning any listening event from up to a full day ago is treated as "listening now." Nothing in the query logic is broken — the comparison, ordering, and per-friend deduplication all work exactly as written. The bug is a mismatched value: a 24-hour window doesn't match the feature's intent (showing who is currently/very recently listening), so any friend whose most recent listen was, say, 20 hours ago still shows up as if they were listening now.

**My fix and side-effect check:** Reduced `RECENT_THRESHOLD` from `timedelta(hours=24)` to `timedelta(minutes=30)`. Verified directly: (1) re-ran the "stale friend" repro after the fix, giving a friend a single event 20 hours old — they correctly no longer appear in `get_friends_listening_now()`; (2) gave a different friend a genuinely recent event (5 minutes old) and confirmed they still appear; (3) confirmed `get_activity_feed()` is unaffected, since it doesn't use `RECENT_THRESHOLD` at all — it still correctly returned older events (including one 20+ hours old) as part of the unfiltered activity history.

---

### Issue #4: Notified when a friend adds my song to a playlist, but not when they rate it

**How I reproduced it:** Identified that the song "Midnight Drive" (id `33ae2afc-b701-4ad2-86fa-6d97ca37ea98`) is shared by user `nova`. Sent `POST /songs/33ae2afc-b701-4ad2-86fa-6d97ca37ea98/rate` as a different user, `darius`, with `{"score": 5}`. The rating was created successfully (`201`-equivalent JSON response with the new `Rating` row). I then called `get_notifications()` for `nova`'s user ID directly and confirmed the only notification present was the pre-existing seed-data `song_added_to_playlist` notification (timestamped from seeding) — no new notification was created for the rating action, even though it happened afterward. This confirms ratings never notify the song's sharer.

**How I found the root cause:** Compared `rate_song()` and `add_to_playlist()` in `notification_service.py` side by side, since both represent "another user interacts with my shared song." `add_to_playlist()` ends with an explicit `create_notification(...)` call (guarded so the sharer doesn't get notified about their own action). `rate_song()` performs its validation, upsert, and commit, and then simply `return`s — there is no call to `create_notification()` anywhere in the function.

**The root cause:** `rate_song()` never invokes `create_notification()`. This isn't a broken condition or typo — the notification step was never written for this code path, even though the identical pattern exists one function above it for playlist additions. Ratings are persisted correctly, but the song's original sharer is never informed.

**My fix and side-effect check:** Added a `create_notification()` call in `rate_song()` after the commit, guarded the same way as `add_to_playlist()` (skip notifying if `rater.id == song.shared_by`). Chose to notify on every call to `rate_song()`, including re-rating (the `existing` branch) — a friend changing their rating is still a new interaction worth surfacing, consistent with how `add_to_playlist()` doesn't special-case repeat adds either. Verified directly: (1) rating someone else's song (`darius` rating `nova`'s "Midnight Drive") produced exactly one new `song_rated` notification for `nova`, with the correct body text and score; (2) rating your own song (`nova` rating her own "Midnight Drive") produced no new notification — confirmed by an unchanged notification count before and after; (3) the existing `song_added_to_playlist` notification path was not touched by this change and still appears correctly in the same notification list.

---

## Commits

All three fixes are committed as separate commits on `bugfix/mixtape`, using conventional commit format:

- `3a435e4` — `fix: remove incorrect Sunday exception blocking streak increment`
- `38f1d38` — `fix: notify song sharer when their song is rated`
- `5b3bef1` — `fix: reduce listening-now threshold from 24 hours to 30 minutes`

## Screenshot

![git log output](Screenshot%202026-07-06%20at%209.55.13%20PM.png)
