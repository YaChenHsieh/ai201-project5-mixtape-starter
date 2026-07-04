## Model (model.py)
There are 8 models, plus 3 association tables.
#### Models: 
1. User — id, username, email, listening_streak, last_listened_at
2. Tag — id, name (no direct relationship column, only via song_tags)
3. Song — shared_by → User (FK)
4. ListeningEvent — user_id → User, song_id → Song
5. Rating — user_id → User, song_id → Song (unique per user+song)
6. Playlist — created_by → User
7. Notification — user_id → User

#### Association tables (many-to-many):
1. friendships — User ↔ User (symmetric self-referential)
2. song_tags — Song ↔ Tag
3. playlist_entries — Playlist ↔ Song (with position, added_by, added_at)

#### Relationship summary:
- User → Song: one-to-many (shared_songs, via Song.shared_by)
- User ↔ User: many-to-many self-referential friendship (friendships)
- User → Rating: one-to-many; Song → Rating: one-to-many (Rating is really the join between User and Song with a score)
- User → ListeningEvent: one-to-many; Song → ListeningEvent: one-to-many (join entity for listening history/feeds)
- User → Notification: one-to-many
- User → Playlist: one-to-many (playlists, via Playlist.created_by)
- Playlist ↔ Song: many-to-many (playlist_entries, ordered)
- Song ↔ Tag: many-to-many (song_tags)

## Router 
There are 4 routers
### feed
This module defines the Flask blueprint for feed-related HTTP endpoints — it's a thin routing layer with no business logic of its own; it just delegates to services/feed_service.py and handles JSON responses/errors.
- listening_now(user_id) — GET /`<user_id>`/listening-now. 
<br>Calls get_friends_listening_now(user_id) to fetch which of the user's friends are currently listening to something, returns {feed, count} as JSON. Returns 404 with an error message if the service raises ValueError (e.g., user not found).

- activity(user_id) — GET /`<user_id>`/activity. 
<br>Calls get_activity_feed(user_id) to fetch the user's friends' recent listening history, returns {feed, count} as JSON. Same 404-on-ValueError handling.

### playlists
This module defines the Flask Blueprint routes for managing playlists in a music streaming or sharing application ("Mixtape"). It acts as the API controller layer, handling incoming HTTP requests, extracting payloads, delegating the core business logic to underlying services (playlist_service and notification_service), and returning JSON responses with appropriate HTTP status codes.
- create()
<br>Calls create_playlist() from the service layer and returns the newly created playlist object as JSON with a 201 Created status code.

- get_detail(playlist_id)
<br> Returns the playlist details as JSON. If the playlist doesn't exist, catches a ValueError and returns a 404 Not Found error.

- get_songs(playlist_id)
<br> Retrieves all the songs contained within a specific playlist.
Returns a JSON object containing an array of songs and a total count of those songs. If the playlist isn't found, it returns a 404 Not Found error.

- add_song(playlist_id)
<br> Accepts a JSON payload with song_id and added_by to append a new track to the specified playlist. 
    * Validates that both fields are present in the request payload (400 Bad Request if missing).
    * Delegates the action to add_to_playlist() from the notification service Returns a 201 Created confirmation message on success.

### songs
Define the Flask blueprint for song-related HTTP endpoints — search, retrieval, rating, and listening. Like feed.py, it's a thin routing layer that validates request input and delegates to services.

- search(query) — GET /search?q=.... Requires a q query param (400 if missing). 
<br>Calls search_songs(query) from services/search_service.py, returns {results, count}.

- get_song_detail(song_id) — GET /<song_id>. 
<br>Calls get_song(song_id) to fetch a single song's details. Returns 404 if not found (ValueError).

- rate(song_id) — POST /<song_id>/rate. Reads user_id and score from JSON body (400 if either missing). 
<br>Calls rate_song(user_id, song_id, score) from services/notification_service.py, returns the created/updated rating (201) or 400 on ValueError.

- listen(song_id) — POST /<song_id>/listen. Reads user_id from JSON body (400 if missing). 
<br>Calls record_listening_event(user_id, song_id) from services/streak_service.py — this is the entry point for the feed/streak flow traced earlier. Returns the created ListeningEvent (201) or 400 on ValueError.

### users
routes/users.py is the Flask blueprint for user account, streak, and notification endpoints — again a thin routing layer, with one exception: get_user queries the User model directly instead of going through a service.
- get_user(user_id) — GET /<user_id>. 
<br>Directly does db.session.get(User, user_id) (no service layer), returns user.to_dict() or 404 if not found.

- streak(user_id) — GET /<user_id>/streak. 
<br>Calls get_streak(user_id) from services/streak_service.py, returns {user_id, streak}. 404 on ValueError.

- notifications(user_id) — GET /<user_id>/notifications. Reads optional unread_only query param. 
<br>Calls get_notifications(user_id, unread_only=...) from services/notification_service.py, returns {notifications, count}. 404 on ValueError.

- read_notification(notification_id) — POST /notifications/<notification_id>/read. 
<br>Calls mark_as_read(notification_id) from services/notification_service.py, returns a success message. 404 on ValueError.

## Data Flow
Flow: song → feed

1. routes/songs.py:listen() — POST endpoint receives user_id/song_id.
2. → services/streak_service.py:record_listening_event(user_id, song_id):
    - validates user via db.session.get(User, ...)
    - creates a ListeningEvent row, db.session.add()s it
    - → update_listening_streak(user, now) updates streak fields
    - db.session.commit() persists both
3. Route returns event.to_dict().

#### Sol Process

### Problem 1: My Listening Streak Keeps Resetting

#### Reproduce

1. Query the `user` table to check the current listening streak and the last listening date:

   ```bash
   sqlite3 instance/mixtape.db 'SELECT id, username, listening_streak, last_listened_at FROM user;'
   ```

2. Query the `song` table to get a valid song ID:

   ```bash
   sqlite3 instance/mixtape.db 'SELECT id, title FROM song LIMIT 5;'
   ```

3. Record a listening event for a user who has not listened today:

   ```bash
   curl -X POST http://127.0.0.1:5000/songs/<song-id>/listen \
   -H 'Content-Type: application/json' \
   -d '{"user_id":"<user-id>"}'
   ```

4. Query the `user` table again and check whether `listening_streak` and `last_listened_at` were updated correctly:

   ```bash
   sqlite3 instance/mixtape.db 'SELECT id, username, listening_streak, last_listened_at FROM user;'
   ```

5. Observe that the streak may incorrectly reset to `1` when the weekday condition fails. The original condition specifically fails on Sunday. To reproduce it on another day during testing, temporarily change the weekday value in the condition.

#### Root Cause

The condition for the "listened yesterday" case also checks the current weekday:

```python
and today.weekday() != 6
```

This condition is unrelated to whether the user listened yesterday. When the condition evaluates to `False`, the code skips the increment branch and resets the listening streak.

#### Fix

Remove the weekday condition so the streak logic only compares listening dates:

```python
if last_listened_date == yesterday:
    user.listening_streak += 1
```

This allows the streak to increment correctly on every day of the week.

---

### Problem 2: Friends Listening Now Shows People from Yesterday

#### Reproduce

1. Query the `user` table and identify a user whose friend has a `last_listened_at` timestamp from yesterday:

   ```bash
   sqlite3 instance/mixtape.db 'SELECT * FROM user;'
   ```

2. Call the Friends Listening Now endpoint:

   ```bash
   curl http://127.0.0.1:5000/feed/<user-id>/listening-now
   ```

3. Check the response. A friend whose `last_listened_at` value is from yesterday is incorrectly included in the feed.

#### Root Cause

The service only subtracts `RECENT_THRESHOLD` from the current time. For example, a 24-hour threshold can include listening events from yesterday, even though the endpoint is intended to show friends who listened today.

#### Fix

Calculate both the recent-time cutoff and the beginning of today, then use whichever cutoff is later:

```python
now = datetime.now(timezone.utc)
today_start = now.replace(hour=0, minute=0, second=0, microsecond=0)
recent_cutoff = now - RECENT_THRESHOLD
cutoff = max(today_start, recent_cutoff)
```

Then filter listening events using the final cutoff:

```python
ListeningEvent.listened_at >= cutoff
```

This ensures that returned events are both recent enough and from the current day.
