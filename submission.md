# Mixtape — Project 5 Submission

---

## Codebase Map

This map describes how the app is organized before bug fixes. It is meant to show that I read the code and understand how data moves through the system.

### What the app does

Mixtape is a Flask API for a social music app. Users share songs, rate them, listen (which updates streaks), build collaborative playlists, get notifications when friends interact with their songs, and view friend activity feeds. All data lives in SQLite via SQLAlchemy.

### Main files and roles

| File / folder | Role |
|---------------|------|
| `app.py` | Application factory (`create_app`). Creates the Flask app, configures the DB (`sqlite:///mixtape.db`), initializes SQLAlchemy, registers route blueprints under `/songs`, `/playlists`, `/users`, `/feed`, and runs `db.create_all()` on startup. |
| `models.py` | All database tables as SQLAlchemy models: `User`, `Song`, `Tag`, `Rating`, `ListeningEvent`, `Playlist`, `Notification`. Also defines join tables `friendships`, `song_tags`, and `playlist_entries` (playlist songs with a `position` column for ordering). |
| `routes/` | HTTP layer only. Each file defines a Flask blueprint. Handlers parse request params/body, call a service function, and return JSON (status codes for errors). No business logic here. |
| `routes/songs.py` | Search, get song detail, rate a song, record a listen event. |
| `routes/playlists.py` | Create playlist, get playlist metadata, list songs in a playlist, add a song to a playlist. |
| `routes/users.py` | Get user profile, get listening streak, list notifications, mark notification read. |
| `routes/feed.py` | Friends “listening now” feed and general friend activity feed. |
| `services/` | All business logic. Routes delegate here. This is where the five open issues live. |
| `services/streak_service.py` | Records listens and updates `User.listening_streak` / `last_listened_at`. |
| `services/feed_service.py` | Builds friend listening feeds from `ListeningEvent` rows. |
| `services/search_service.py` | Song search by title/artist; single-song lookup. |
| `services/notification_service.py` | Creates and reads notifications; also handles “add song to playlist” side effects. |
| `services/playlist_service.py` | Create playlist, get playlist metadata, get ordered songs in a playlist. |
| `seed_data.py` | Wipes and repopulates the DB with 5 users, 13 songs, 3 playlists, friendships, listening events, and sample notifications. Run with `python seed_data.py`. |
| `tests/` | Pytest tests for streaks, search, and playlists. |

### Data model (short)

- **User** — account, streak fields, friends (many-to-many via `friendships`).
- **Song** — a shared track (`shared_by` FK → User). Not a global catalog row; each share is its own song record.
- **Rating** — user + song + score (1–5), unique per user/song pair.
- **ListeningEvent** — user listened to a song at a timestamp (drives streaks and feeds).
- **Playlist** — named collection; songs linked through `playlist_entries` with `position` for order.
- **Notification** — message to a user (`notification_type`, `body`, `read` flag).

Relationships like `user.shared_songs` are Python shortcuts; the real FK is `song.shared_by`.

### Data flow — friend adds your song to a playlist (notification)

This traces a feature end-to-end and shows how routes, services, models, and notifications connect.

1. **Request:** `POST /playlists/<playlist_id>/songs` with JSON `{ "song_id", "added_by" }`.
2. **Route:** `routes/playlists.py` → `add_song()` validates `song_id` and `added_by`, then calls `notification_service.add_to_playlist(playlist_id, song_id, added_by)`.
3. **Service — load entities:** `add_to_playlist()` loads `Song`, `User` (adder), and `Playlist` from the DB. Raises `ValueError` if any missing (route returns 400).
4. **Service — write playlist:** If the song is not already in `playlist.songs`, append it and `commit` (writes a row to `playlist_entries`).
5. **Service — notify sharer:** If `song.shared_by != added_by`, call `create_notification()` with type `song_added_to_playlist` and a human-readable `body` naming the adder, song title, and playlist name.
6. **Service — persist notification:** `create_notification()` inserts a `Notification` row and commits.
7. **Response:** Route returns `201` with `{ "message": "Song added to playlist" }`.
8. **Read back:** Sharer can `GET /users/<sharer_id>/notifications` → `get_notifications()` → list of notification dicts ordered by `created_at` desc.

A similar pattern exists for rating (`POST /songs/<id>/rate` → `rate_song()`), but notification creation is only wired up for the playlist path in the starter code.

### Data flow — user listens to a song (streak)

1. `POST /songs/<song_id>/listen` with `{ "user_id" }` → `routes/songs.py` → `streak_service.record_listening_event()`.
2. Creates a `ListeningEvent`, calls `update_listening_streak()` (compares today vs `last_listened_at` to increment or reset streak), commits.
3. Streak readable via `GET /users/<user_id>/streak` → `get_streak()`.

### Patterns in how the app is organized

1. **Thin routes, fat services** — Every route does input parsing + JSON response; logic lives in `services/`.
2. **Issue → service mapping** — README lists which service file owns each feature area; tracing a broken endpoint means following the import from the route into that service.
3. **SQLite + SQLAlchemy** — No raw SQL in routes; queries use `db.session` and model classes.
4. **Join tables for many-to-many** — Tags on songs (`song_tags`), songs in playlists (`playlist_entries` with order), friendships (`friendships`).
5. **Errors as `ValueError`** — Services raise `ValueError` with a message; routes catch and map to 400/404 JSON errors.
6. **Seed-driven development** — `seed_data.py` creates realistic IDs and relationships; reproducing bugs often means knowing what the seed script puts in the DB (e.g. which songs have multiple tags, how many songs per playlist).

---

## Root Cause Analyses

### Issue #5: The last song in a playlist never shows up

**How I reproduced it**

1. Ran `python seed_data.py` and started the app with `flask --app app:create_app run`.
2. Looked up the playlist ID for "Late Night Vibes" in the database (seeded with `all_songs[:7]` in `seed_data.py`, so 7 songs are stored in `playlist_entries`).
3. Called `GET /playlists/<playlist_id>/songs`.
4. The response returned `"count": 6` every time, regardless of how many songs were actually in the playlist. The missing song was always the last one by position (e.g. "Free Throws" for Late Night Vibes).

**How I found the root cause**

1. README maps Issue #5 to `playlist_service.py`.
2. Opened `routes/playlists.py` → `get_songs()` calls `get_playlist_songs(playlist_id)` and returns the list with `len(songs)` as count.
3. Opened `get_playlist_songs()` in `services/playlist_service.py`. The SQLAlchemy query (join on `playlist_entries`, filter by playlist, order by `position` ASC) looked correct.
4. The bug was on the return line: `return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice drops the final element of every result set. I confirmed in flask shell that `len(p.songs)` was 7 while the API returned 6.

**Root cause**

`get_playlist_songs()` fetched all songs in the correct order but returned `songs[:-1]` instead of `songs`. In Python, `[:-1]` excludes the last item in a list. So a playlist with N songs always returned N−1 — specifically, the song at the highest `position` value was silently dropped. The query and join table were fine; the data was lost in the return statement.

**Fix and side-effect check**

- **Fix:** Changed the return to `return [song.to_dict() for song in songs]` (removed `[:-1]`).
- **Verify:** Re-ran `GET /playlists/<id>/songs` for "Late Night Vibes" → `"count": 7`, all seven seeded titles present including "Free Throws" (last position).
- **Side effects:** Confirmed `GET /playlists/<id>` (metadata only, uses `get_playlist()`) still works. Checked that other seeded playlists ("Friday Energy", "Study Mode") return their full song counts. Had to kill duplicate Flask processes on port 5000 so the restarted server actually loaded the updated code.

### Issue #4: No notification when a friend rates your song

**How I reproduced it**

1. Read `seed_data.py` — simone (`users[2]`) shared "Crown Heights Anthem"; darius is a different user.
2. Used `flask shell` to get IDs for simone, darius, and the song.
3. `GET /users/<simone_id>/notifications` → `"count": 0` (no `song_rated` notification).
4. `POST /songs/<song_id>/rate` with body `{"user_id": "<darius_id>", "score": 5}` → returned `201` and saved the rating successfully.
5. `GET /users/<simone_id>/notifications` again → still no new notification. Rating worked; notify did not.

**How I found the root cause**

1. README maps Issue #4 to `notification_service.py`. Instructions hint: compare the working notification path to the broken one.
2. Traced `POST /songs/<id>/rate` → `routes/songs.py` → `rate_song()`.
3. Read `rate_song()` — it validates, upserts a `Rating`, commits, and returns. No call to `create_notification()`.
4. Opened `add_to_playlist()` in the same file — after saving the playlist change, it calls `create_notification()` for `song.shared_by` when the adder is not the sharer (lines 64–70).
5. Confirmed seed data already includes a working `song_added_to_playlist` notification for nova, proving the notification system works — the rating path was simply never wired up.

**Root cause**

`rate_song()` saved the rating correctly but never created a notification for the song's original sharer. This was not a typo in the rating upsert logic (`existing` is a `Rating` object from `.first()`, not a bool — that block is correct). The bug was architectural: `add_to_playlist()` included a `create_notification()` step for `song.shared_by`, but `rate_song()` was implemented without the equivalent step. So when a friend rated someone else's shared song, the `rating` table updated but the `notification` table never got a row.

**Fix and side-effect check**

- **Fix:** After `db.session.commit()` in `rate_song()`, added the same pattern as `add_to_playlist()`: if `song.shared_by != user_id`, call `create_notification()` with type `song_rated` and a message naming the rater, song title, and score.
- **Verify:** Restarted Flask, repeated the rate POST, then `GET /users/<simone_id>/notifications` → new `song_rated` notification appears.
- **Side effects:** Self-rating (sharer rates own song) should not notify — guarded by `song.shared_by != user_id`. Existing playlist-add notifications unchanged. Rating create/update logic unchanged.

### Issue #2: Friends Listening Now shows stale listeners

**How I reproduced it**

1. README maps Issue #2 to `feed_service.py`. Seed data comments say recent events are ~10–20 minutes old and older events start at 2+ hours ago.
2. Used `flask shell` to get nova's user id (nova is friends with darius, simone, and kenji per `seed_data.py`).
3. `GET /feed/<nova_id>/listening-now` returned friends with `listened_at` timestamps hours in the past (e.g. kenji ~3 hours ago), not people listening right now.
4. The filter logic (`listened_at >= cutoff`) was passing those events because the cutoff window was 24 hours wide.

**How I found the root cause**

1. Traced `GET /feed/<user_id>/listening-now` → `routes/feed.py` → `get_friends_listening_now()`.
2. Found `RECENT_THRESHOLD = timedelta(hours=24)` at the top of `feed_service.py`.
3. `cutoff = now - RECENT_THRESHOLD` means any listen in the last 24 hours qualifies — including yesterday evening.
4. The query and dedup logic were correct; the threshold constant was wrong for a feature named "Listening Now". Compared to `get_activity_feed()` which intentionally has no recency filter — confirming this function was meant to be stricter.

**Root cause**

`RECENT_THRESHOLD` was set to 24 hours, so `get_friends_listening_now()` treated any friend who listened in the past day as "listening now." The filter comparison (`listened_at >= cutoff`) worked as written, but the cutoff was far too loose for the product intent. Friends who listened hours ago still appeared in the feed.

**Fix and side-effect check**

- **Fix:** Changed `RECENT_THRESHOLD` from `timedelta(hours=24)` to `timedelta(minutes=30)` to match the seed data comment that recent events are within the past 30 minutes.
- **Verify:** Restarted Flask, re-ran `GET /feed/<nova_id>/listening-now` — stale hours-old entries dropped off. After `POST /songs/<id>/listen` for a friend, that friend reappears with a fresh `listened_at`.
- **Side effects:** `GET /feed/<user_id>/activity` uses `get_activity_feed()` which has no recency filter — still returns older events. Unchanged by this fix.

---

## AI Usage

I used Cursor (AI-assisted IDE) throughout this project. Below is what it actually helped with and where I still had to verify things myself.

**Use 1 — Building the codebase map**

I asked the AI to walk through how the app is organized: which files own which features, and how a request flows from a route into a service. It pointed me to `routes/playlists.py` → `playlist_service.get_playlist_songs()` as the path for playlist songs, and `notification_service.add_to_playlist()` as the working notification example. That gave me a starting map, but I read each file myself and edited the map so it described the starter code, not the fixed code.

**Use 2 — Tracing bugs from endpoint to root cause**

For Issue #5, I described the symptom (playlist always missing the last song) and the AI suggested following the README’s service mapping into `get_playlist_songs()`. It highlighted the `songs[:-1]` slice on the return line. I did not take that at face value — I ran `flask shell`, loaded the "Late Night Vibes" playlist, and confirmed `len(p.songs)` was 7 while `GET /playlists/<id>/songs` returned `count: 6`. That proved the query was fine and the slice was dropping the last row.

For Issue #4, the AI compared `rate_song()` to `add_to_playlist()` in the same file and noticed only the playlist path called `create_notification()`. I verified by rating simone’s song as darius and checking simone’s notifications before and after — rating saved, notification did not, until I added the missing call.

For Issue #2, the AI traced `GET /feed/.../listening-now` to `RECENT_THRESHOLD` in `feed_service.py`. I confirmed the seed data had listens hours old still showing up, then re-tested after changing the window to 30 minutes.

**Where I course-corrected**

Early on the AI floated Issue #3 (search duplicates) and suggested reproducing with `?q=rap`. I checked `search_service.py` and saw search only matches title/artist, not tags — so that repro did not match the real bug. I skipped #3 and fixed #2, #4, and #5 instead.

I also had to restart Flask and kill duplicate processes on port 5000 before fixes showed up in curl. The AI reminded me of that, but I confirmed it myself when the API kept returning old behavior after a code change.

**What I did not outsource**

I ran all repro steps (`curl`, `flask shell`, `seed_data.py`), wrote the RCA text in my own words from what I observed, and split fixes into separate commits on `bugfix/mixtape` so the git log matches one fix per commit.

