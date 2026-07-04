# Mixtape Bug Hunt — Submission

## AI Usage
I used Claude throughout codebase navigation and debugging. For each bug, I pasted the relevant service file and asked Claude to help me trace the logic and identify suspicious code, rather than asking it to find the bug blind. Specifically:
- For Issue #1 (streak), Claude pointed me to the `days_since_last == 1 and today.weekday() != 6` condition; I verified this myself by running a reproduction in `flask shell` with a simulated Sunday date before accepting it as the root cause.
- For Issue #3 (duplicate search results), Claude's first hypothesis (join fan-out per tag) was wrong — I verified this myself by running the raw SQL query and checking `len(results)`, which showed only 1 row instead of the predicted 3. We were not able to confirm a root cause for this issue in the time available, so I did not include it in my final 3.
- For Issue #5 (playlist), Claude identified the `songs[:-1]` slice on sight; I confirmed by comparing the count returned by `get_playlist_songs()` against the raw `playlist_entries` row count for the same playlist (6 vs 7).
- For Issue #4 (notifications), Claude had me compare `add_to_playlist()` against `rate_song()` line-by-line to see that the notification call was simply missing; I verified with a before/after notification count check in `flask shell`.

In all cases, I ran the reproduction scripts and confirmed the before/after behavior myself rather than trusting Claude's explanation alone. Claude did not write final code for me without me confirming the diagnosis against actual output first.

## Codebase Map

### Main files and their roles
- `app.py` — Flask application factory (`create_app`). Sets up SQLAlchemy config, registers all blueprints with their URL prefixes (`/songs`, `/playlists`, `/users`, `/feed`), and creates DB tables on startup.
- `models.py` — Defines all SQLAlchemy models: `User`, `Song`, `Tag`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, plus association tables `friendships`, `song_tags`, and `playlist_entries` (the last of which stores explicit `position`, `added_by`, and `added_at` for ordering songs within a playlist).
- `routes/` — Thin endpoint layer. `songs.py` handles search, song detail, rating, and listen events; each route parses the request and delegates immediately to a service function.
- `services/` — All business logic lives here. `streak_service.py` handles listening streak increment/reset logic; `search_service.py` handles song search by title/artist; `notification_service.py` handles creating and retrieving notifications plus playlist-add and rating actions; `playlist_service.py` handles playlist creation and ordered song retrieval.

### Data flow: a user rates a song
1. Client sends `POST /songs/<song_id>/rate` with `user_id` and `score` in the JSON body.
2. `routes/songs.py`'s `rate()` function parses the body and calls `notification_service.rate_song(user_id, song_id, score)`.
3. `rate_song()` validates the score, looks up the song and rater, creates or updates a `Rating` row, and commits it. (Originally this function stopped here — see Issue #4 below.)

### Patterns noticed
- Every route delegates immediately to a service function; routes only handle parsing and JSON formatting, all logic lives in `services/`.
- Notification-triggering actions (e.g. `add_to_playlist`) follow a consistent pattern: perform the action, then check if the actor is different from the song's original sharer, then call `create_notification()`. This pattern was not consistently applied to every action that should notify (see Issue #4).

---

## Root Cause Analysis Entries

### Issue #1: My listening streak keeps resetting

**How you reproduced it**
Used `flask shell` to set a test user's `listening_streak` to 5 and `last_listened_at` to one day before "now," then called `update_listening_streak()` with a simulated "now" set to a Sunday (July 5, 2026). Expected the streak to increment to 6 since it was a consecutive day; instead it printed 1.

**How you found the root cause**
- Files checked: `services/streak_service.py`
- Navigation path: Traced `record_listening_event()` → `update_listening_streak()`, then read the if/elif/else block controlling streak changes line by line.
- Moment of confidence: Spotted `elif days_since_last == 1 and today.weekday() != 6:` — an unexplained extra condition on a day-of-week check inside what should be a pure "was it consecutive" check.

**The root cause**
`datetime.weekday()` returns `6` for Sunday. The increment branch required `days_since_last == 1 AND today.weekday() != 6`, meaning a genuinely consecutive listening day was excluded from incrementing if that day happened to be a Sunday, causing the streak to fall through to the `else` branch and reset to 1.

**Your fix and side-effect check**
- Change made: Removed `and today.weekday() != 6` from the `elif` condition, leaving `elif days_since_last == 1:`.
- Why it fixes root cause: The increment branch now fires for any genuinely consecutive day, regardless of weekday.
- Related functionality checked: Confirmed the `days_since_last == 0` (same-day, no change) and the `else` (reset to 1) branches don't reference weekday and are unaffected.

---

### Issue #5: The last song in a playlist never shows up

**How you reproduced it**
Called `get_playlist_songs()` on a test playlist via `flask shell` — got 6 songs back. Cross-checked against the raw `playlist_entries` table for that same playlist, which had 7 rows. The highest-position song was missing from the output.

**How you found the root cause**
- Files checked: `services/playlist_service.py`
- Navigation path: Read `get_playlist_songs()` top to bottom; the query itself (join + filter + order by position) looked correct.
- Moment of confidence: The return line `[song.to_dict() for song in songs[:-1]]` — the `[:-1]` slice drops the last element of an already correctly-ordered list.

**The root cause**
The function correctly queries and orders all songs in the playlist by position, but the final return statement slices the resulting list with `[:-1]`, unconditionally discarding the last (highest-position) song regardless of playlist length or content.

**Your fix and side-effect check**
- Change made: Removed the `[:-1]` slice so the full list is returned: `[song.to_dict() for song in songs]`.
- Why it fixes root cause: No songs are discarded; all songs in the playlist, including the last one, are now returned in position order.
- Related functionality checked: Confirmed `get_playlist()` (metadata only) and `get_user_playlists()` don't touch this query or slicing logic.

---

### Issue #4: I got notified when a friend added my song to a playlist but not when they rated it

**How you reproduced it**
Called `rate_song()` directly via `flask shell` on a song not shared by the rater, then checked `get_notifications()` for the song's original sharer before and after. Notification count stayed at 1 both times — no new notification appeared despite a new rating being saved.

**How you found the root cause**
- Files checked: `services/notification_service.py`
- Navigation path: Compared the working `add_to_playlist()` function against `rate_song()` line-by-line, since both should trigger notifications to the song's original sharer.
- Moment of confidence: `add_to_playlist()` ends with an `if song.shared_by != added_by_user_id: create_notification(...)` block; `rate_song()` has no equivalent call anywhere in the function — it just commits the rating and returns.

**The root cause**
This is architectural, not a typo or bad condition: the notification step present in `add_to_playlist()` was simply never written for the rating path. `rate_song()` persists the `Rating` row correctly but never notifies the song's original sharer that a rating occurred.

**Your fix and side-effect check**
- Change made: Added a `create_notification()` call at the end of `rate_song()`, guarded by `if song.shared_by != user_id`, using `notification_type="song_rated"`, following the same pattern as `add_to_playlist()`.
- Why it fixes root cause: The missing notification step now exists, matching the established pattern used elsewhere in the file.
- Related functionality checked: Reran the reproduction — notification count went from 1 to 2 after rating. Confirmed `add_to_playlist()` and `get_notifications()` are untouched and unaffected.

---

## git log Screenshot
<!-- Paste screenshot of `git log --oneline` on bugfix/mixtape here -->