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