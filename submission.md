# Codebase Map

Before looking into any of the reported issues, I spent some time understanding how the application is organized and how requests flow through the different layers.

### Main files and their responsibilities

`app.py` is the application's entry point. It creates the Flask application, loads the configuration, initializes SQLAlchemy, registers the blueprints for the Songs, Playlists, Users, and Feed modules, and creates the database tables during startup.

`models.py` defines all of the application's database models and relationships. The main entities are `User`, `Song`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`, and `Tag`. It also defines the association tables used for many-to-many relationships, including friendships between users, song tags, and playlist entries. One thing I noticed is that playlist ordering is not based on insertion order. Instead, the `playlist_entries` association table stores a separate `position` field along with information about who added the song and when it was added.

The `routes` folder contains the API endpoints. These files mainly validate request data, call the appropriate service function, and return JSON responses. They contain very little business logic.

* `routes/songs.py` handles song search, song details fetch, song ratings, and listening events.
* `routes/playlists.py` handles playlist creation, retrieving playlists, viewing playlist songs, and adding songs to playlists.
* `routes/users.py` handles user information, listening streaks, notifications, and marking notifications as read.
* `routes/feed.py` handles the Friends Listening Now feed and the activity feed.

The `services` folder contains the application's business logic. This is where most database queries, calculations, and feature-specific processing happen.

* `streak_service.py` manages listening events and updates listening streaks.
* `feed_service.py` generates the Friends Listening Now feed and the general activity feed.
* `search_service.py` performs song search and retrieves song details.
* `notification_service.py` creates notifications, records song ratings, handles playlist notification logic, retrieves notifications, and marks notifications as read.
* `playlist_service.py` creates playlists and retrieves playlist metadata and playlist songs.

`seed_data.py` populates the database with sample users, songs, playlists, friendships, listening events, ratings, and notifications so the application has realistic data for testing.

The `tests` folder contains automated checks for streak logic, song search behavior, and playlist retrieval. These tests reflect intended behavior and are useful later for confirming that fixes do not break related functionality.

### Architecture and organization

The application follows a consistent Route → Service → Model architecture.

The routes are responsible for handling HTTP requests, validating inputs, and formatting responses. Almost every route delegates the actual work to a service function. The services contain the business logic and interact with the database through the SQLAlchemy models. The models define the application's data structure and relationships.

Because every feature follows this same pattern, once I identified the route for a feature, it was straightforward to trace execution into the corresponding service and then to the relevant database models.

### Example data flow: Adding a song to a playlist

When a user adds a song to a playlist, the request first reaches `POST /playlists/<playlist_id>/songs` in `routes/playlists.py`.

The route validates that both `song_id` and `added_by` are present in the request before calling `add_to_playlist()` in `notification_service.py`.

Inside `add_to_playlist()`, the service retrieves the `Song`, `User`, and `Playlist` records from the database. If the song is not already part of the playlist, it is added to the playlist and the database is updated. After that, if the user adding the song is different from the user who originally shared it, the service creates a new `Notification` record so the original sharer is informed that their song was added to a playlist.

The overall execution flow for this feature is:

**HTTP Request → Route → Service → SQLAlchemy Models → Database → JSON Response**

### Overall observations

One pattern I noticed throughout the project is that the routes remain intentionally thin, while the service layer contains almost all of the application logic. This separation makes the codebase easier to navigate because each layer has a clear responsibility. After identifying the correct route, I could consistently follow the same execution path into the service layer and then to the underlying models and database operations.

---

# Root Cause Analysis

## Issue #5: The last song in a playlist never shows up

### How I reproduced it

I first seeded the database and started the Flask application. Since there wasn't an endpoint to list all playlists, I opened the Flask shell and queried the `Playlist` model to retrieve a valid playlist ID.

Using that playlist ID, I sent a `GET /playlists/<playlist_id>/songs` request. The API returned 6 songs for the playlist.

To verify whether the issue was with the database or the application's retrieval logic, I opened the Flask shell again and queried the same playlist using SQLAlchemy. I retrieved the songs associated with the playlist through the `playlist_entries` association table and confirmed that the database actually contained 7 songs in the correct order. Since the database contained all 7 songs but the API returned only 6, I confirmed that the last song was being omitted during the retrieval process rather than being missing from the database.

### How I found the root cause

I started from the `GET /playlists/<playlist_id>/songs` route in `routes/playlists.py` and followed the call to `get_playlist_songs()` in `playlist_service.py`. The database query correctly retrieved all songs for the playlist and ordered them using the `position` column from the `playlist_entries` association table.

The point that confirmed the root cause was the return statement. Although the query returned the complete list of songs, the function converted only `songs[:-1]` into the response, which excluded the final song before returning the result.

### The root cause

The database query was returning all songs in the playlist correctly, but the final return statement sliced the list using `songs[:-1]`. In Python, this slice returns every element except the last one. As a result, the API always omitted the final song in the playlist even though it existed in the database and had been retrieved successfully.

### Your fix and side-effect check

I removed the unnecessary list slicing and returned the complete list of songs retrieved by the query. This ensured that every song in the playlist was included in the API response while preserving the existing ordering logic.

After making the change, I called the `GET /playlists/<playlist_id>/songs` endpoint again and confirmed that it now returned all 7 songs in the playlist. I also ran the playlist test suite, and all tests passed successfully, confirming that playlist ordering and empty playlist behavior were not affected by the fix.
