# **MediaSession-CS Framework**

- C/S Architecture
  - Defines a standard communication interface between the UI and the player in audio/video applications, achieving complete decoupling between the UI and the player.

![ts8v1qwi2k](Brief Introduction to MediaSession and Command Transmission and Support_imgs\dsUvjUo8wYG.png)

## **MediaSession (Server) – Controlled Player Endpoint**

- Manages all communication with the `MediaPlayer` and hides its APIs from other parts of the application. The system can only access the player through the `MediaSession`.
- Maintains information such as:
  - **PlaybackState**: Player status (e.g., play/pause).
  - **MetaData**: Related information about the content being played (e.g., title/artist).
- The `MediaSession` can receive callbacks from one or more `MediaController` instances. The response logic to these callbacks must remain consistent, regardless of which client application initiates them.

## **MediaController (Client)**

- The UI communicates solely with the `MediaController`. It converts transport control operations into callbacks to the `MediaSession`. Whenever the session state changes, the `MediaController` receives callbacks from the `MediaSession` and can update the UI accordingly.
- A `MediaController` can connect to only one `MediaSession` at a time.
- Creating a `MediaController` requires a pairing token from the controlled endpoint, which means it should be created only after a successful browser connection.

------

# **MediaSession Authorization Framework**

## **MediaBrowser (Client)**

- Used to connect to the `MediaBrowserService` and subscribe to data. It allows retrieval of the service connection state and music data via registered callbacks. Typically created on the client side.

## **MediaBrowserService (Server)**

- Contains two key callbacks:
  - `onGetRoot`: Manages client connection requests and determines whether the client is allowed to connect.
  - `onLoadChildren`: Triggered when a `MediaBrowser` sends a data subscription request. This is typically where asynchronous data fetching is performed before sending the data back through the `MediaBrowser`'s registered interface.



# **Command Transmission**

![image-2024-9-20_16-15-7](Brief Introduction to MediaSession and Command Transmission and Support_imgs\Apm71s0eRqA.png)

## **Client-to-Server Communication**

| Action             | `MediaController.TransportControls` | `MediaSession.Callback`             |
| ------------------ | ----------------------------------- | ----------------------------------- |
| Play               | `play()`                            | `onPlay()`                          |
| Pause              | `pause()`                           | `onPause()`                         |
| Stop               | `stop()`                            | `onStop()`                          |
| Seek to Position   | `seekTo(long)`                      | `onSeekTo(long)`                    |
| Fast Forward       | `fastForward()`                     | `onFastForward()`                   |
| Rewind             | `rewind()`                          | `onRewind()`                        |
| Next               | `skipToNext()`                      | `onSkipToNext()`                    |
| Previous           | `skipToPrevious()`                  | `onSkipToPrevious()`                |
| Set Playback Speed | `setPlaybackSpeed(float)`           | `onSetPlaybackSpeed(float)`         |
| Rate               | `setRating(Rating)`                 | `onSetRating(Rating)`               |
| Send Custom Action | `sendCustomAction(String,Bundle)`   | `onSendCustomAction(String,Bundle)` |

------

## **Server-to-Client Callbacks**

| Action                 | `MediaSession`                    | `MediaController.Callback`              |
| ---------------------- | --------------------------------- | --------------------------------------- |
| Current Track Metadata | `setMetadata(MediaMetadata)`      | `onMetadataChanged(MediaMetadata)`      |
| Playback State         | `setPlaybackState(PlaybackState)` | `onPlaybackStateChanged(PlaybackState)` |
| Playback Queue         | `setQueue(List QueueItem)`        | `onQueueChanged(List QueueItem)`        |
| Queue Title            | `setQueueTitle(CharSequence)`     | `onQueueTitleChanged(CharSequence)`     |
| Extras                 | `setExtras(Bundle)`               | `onExtrasChanged(Bundle)`               |

------

# **Command Support**

## **Topic 1: Can you determine if the player supports specific commands via code?**

**Conclusion**: You can determine support for non-custom commands, but not for custom commands.

### Example: Checking Supported Non-Custom Commands

```
java复制代码MediaController mediaController = getMediaController(context);
if (mediaController != null) {
    PlaybackState playbackState = mediaController.getPlaybackState();

    if (playbackState != null) {
        long actions = playbackState.getActions();

        if ((actions & PlaybackState.ACTION_PLAY) != 0) {
            Log.d("MediaSession", "Player supports Play command.");
        }
        if ((actions & PlaybackState.ACTION_PAUSE) != 0) {
            Log.d("MediaSession", "Player supports Pause command.");
        }
        if ((actions & PlaybackState.ACTION_FAST_FORWARD) != 0) {
            Log.d("MediaSession", "Player supports Fast Forward command.");
        }
    }
}
```

------

## **Topic 2: Can you determine if the player successfully executed a command via code?**

**Conclusion**: You can only determine the success of certain commands (e.g., play/pause) using `PlaybackState`, but there are no callbacks for commands like next/previous.

### Example: Checking Command Execution Success

```
java复制代码// Assume a MediaController instance exists
MediaController mediaController = ...;

// Create and register a Callback to listen for state changes
MediaController.Callback callback = new MediaController.Callback() {
    @Override
    public void onPlaybackStateChanged(PlaybackState state) {
        super.onPlaybackStateChanged(state);
        if (state != null) {
            int playbackState = state.getState();
            switch (playbackState) {
                case PlaybackState.STATE_PLAYING:
                    Log.d("MediaController", "Playback started successfully.");
                    break;
                case PlaybackState.STATE_PAUSED:
                    Log.d("MediaController", "Playback is paused.");
                    break;
                case PlaybackState.STATE_STOPPED:
                    Log.d("MediaController", "Playback stopped.");
                    break;
                case PlaybackState.STATE_ERROR:
                    Log.e("MediaController", "Playback error: " + state.getErrorMessage());
                    break;
                default:
                    Log.d("MediaController", "Playback state changed: " + playbackState);
                    break;
            }
        }
    }
};

// Register the Callback
mediaController.registerCallback(callback);

// Execute play operation
mediaController.getTransportControls().play();

// Unregister the Callback when appropriate
// mediaController.unregisterCallback(callback);
```