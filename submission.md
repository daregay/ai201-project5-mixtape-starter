# Project 5 – Mixtape Bug Hunt

## AI Usage

I used ChatGPT during this project to help me understand the structure of the codebase, explain how different files connected together, and trace the flow of requests from the routes to the service layer. I also used it to explain functions while I investigated bugs. I verified all diagnoses by reading the code and reproducing each bug before making any changes.

---

# Codebase Map

## Main Files

### app.py
Creates the Flask application, configures the SQLAlchemy database, and registers the application's blueprints for songs, playlists, users, and feed routes.

### models.py
Defines all SQLAlchemy database models, including User, Song, Playlist, Rating, ListeningEvent, Notification, and Tag. It also defines the association tables used for friendships, song tags, and playlist entries.

### routes/
The routes receive HTTP requests, validate request data, call the appropriate service function, and return JSON responses. The routes themselves contain very little business logic.

### services/
The services contain the application's business logic. Each service is responsible for one major feature:

- streak_service.py — listening streak logic
- feed_service.py — Friends Listening Now feed
- search_service.py — song searching
- notification_service.py — playlist and rating notifications
- playlist_service.py — playlist creation and retrieval

### seed_data.py
Populates the database with sample users, songs, playlists, and tags for testing.

### tests/
Contains automated tests for streaks, search, and playlists.

---

# Data Flow Example

Example: Rating a Song

1. The client sends a POST request to `/songs/<song_id>/rate`.
2. `routes/songs.py` validates the request and calls `notification_service.rate_song()`.
3. `notification_service.py` validates the user and song, creates or updates the rating, and commits the change to the database.
4. The route returns the rating as JSON.

---

# Architecture Pattern

The application follows a layered architecture:

Client Request
→ Route
→ Service
→ SQLAlchemy Models / Database
→ JSON Response

The routes are intentionally thin and delegate almost all business logic to the service layer.




## Issue #5 – The last song in a playlist never shows up

### How I reproduced it

I sent a GET request to the `GET /playlists/<playlist_id>/songs` endpoint for the seeded "Friday Energy" playlist. The seed data created seven songs for this playlist, but the endpoint only returned six songs.

### How I found the root cause

I started at `routes/playlists.py` and followed the request to `playlist_service.get_playlist_songs()`. After reading the function, I noticed that it queried all songs correctly but sliced the results before returning them.

### The root cause

The function returned `songs[:-1]`, which excludes the last element of the list. Because of this slice, the newest song in every playlist was always omitted from the response.

### My fix and side-effect check

I changed the return statement to return the full `songs` list instead of `songs[:-1]`. After restarting the application, I verified that the playlist endpoint returned all seven songs and confirmed that the playlist ordering remained correct.



## Issue #4 – Rating notifications never appear

### How I reproduced it

I submitted a rating using the `POST /songs/<song_id>/rate` endpoint and then checked the song owner's notifications with `GET /users/<user_id>/notifications`. The rating was saved successfully, but no notification was created.

### How I found the root cause

I started in `routes/songs.py`, which calls `notification_service.rate_song()`. I compared `rate_song()` with `add_to_playlist()`, which successfully generated notifications. I found that `add_to_playlist()` called `create_notification()`, while `rate_song()` never did.

### The root cause

The rating functionality created or updated the `Rating` record and committed it to the database, but it never created a notification for the original song owner. As a result, ratings were stored correctly, but users were never notified.

### My fix and side-effect check

I added a call to `create_notification()` after saving the rating, while checking that users are not notified when rating their own songs. After testing, the song owner correctly received a `song_rated` notification and the rating functionality continued to work normally.



## Issue #1 – My listening streak keeps resetting

### How I reproduced it

Using the Flask shell, I set a user's listening streak to 12 and their last listening date to Saturday. I then called `update_listening_streak()` with a Sunday date. Instead of increasing the streak to 13, the function reset it to 1.

### How I found the root cause

I traced the request from `routes/songs.py` to `streak_service.record_listening_event()`, then into `update_listening_streak()`. The conditional logic contained a special check for Sundays that prevented consecutive Saturday-to-Sunday listening from increasing the streak.

### The root cause

The code only incremented the streak when `days_since_last == 1` **and** the current day was not Sunday (`today.weekday() != 6`). Because Sunday failed this condition, the code always fell into the reset branch even though Saturday and Sunday are consecutive calendar days.

### My fix and side-effect check
I removed the unnecessary Sunday check so the streak increments whenever the user listened exactly one day earlier. The streak still resets to 1 when more than one day has passed without listening. I verified the fix by reproducing the original scenario and confirming that the streak increased from 12 to 13 on Sunday while still resetting correctly after skipped days.