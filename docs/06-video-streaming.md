# Video Streaming

This document explains how video streaming works between the robot and the client.

## Overview

The system uses WebRTC's media streaming capabilities to receive video (and optionally audio) from the robot. The video stream is sent over the WebRTC peer connection separately from the data channel used for commands.

## Video Track Setup

### Browser-Side Initialization

When the WebRTC peer connection is created, video and audio transceivers are added:

```18:22:javascript/go2webrtc.js
    this.pc.addTransceiver("video", { direction: "recvonly" });
    this.pc.addTransceiver("audio", { direction: "sendrecv" });
    this.pc.addEventListener("track", this.trackEventHandler.bind(this));
```

**Key points:**
- **Video**: `recvonly` - Only receive, don't send
- **Audio**: `sendrecv` - Bidirectional audio
- **Track events**: Listener for when tracks arrive

### Track Event Handling

```29:35:javascript/go2webrtc.js
  trackEventHandler(event) {
    if (event.track.kind === "video") {
      this.VidTrackEvent = event;
    } else {
      this.AudTrackEvent = event;
    }
  }
```

When the robot sends media tracks, they're stored for later use.

## Enabling Video Stream

Video isn't automatically sent - you must explicitly request it after validation:

```143:164:javascript/go2webrtc.js
  rtcValidation(msg) {
    if (msg.data === "Validation Ok.") {
      logMessage("Validation OK");
      this.validationResult = "SUCCESS";

      // TODO: execute all the registred callbacks in a map defined
      // in the initRTC function

      // TODO this should be on the callback for video on message
      if (document.getElementById("video-frame")) {
        logMessage("Playing video");
        logMessage("Sending video on message");
        this.publish("", "on", DataChannelType.VID);

        document.getElementById("video-frame").srcObject =
          this.VidTrackEvent.streams[0];
      }
    } else {
      logMessage(`Sending validation key ${msg.data}`);
      this.publish("", encryptKey(msg.data), DataChannelType.VALIDATION); // );
    }
  }
```

**Flow:**
1. After validation succeeds
2. Send video "on" message via data channel
3. Get video element from DOM
4. Assign received video stream to element's `srcObject`

## Video Control Messages

### Turn Video On

```javascript
globalThis.rtc.publish("", "on", DataChannelType.VID);
```

### Turn Video Off

```javascript
globalThis.rtc.publish("", "off", DataChannelType.VID);
```

The empty string `""` for topic indicates this is a direct video control command.

## Python Backend Video Handling

### Media Blackhole (Current Implementation)

Currently, the Python backend doesn't process video:

```88:91:python/go2_webrtc/go2_connection.py
        # self.audio_track = Go2AudioTrack()
        # self.video_track = Go2VideoTrack()
        # self.video_track = Go2CvVideo()
        self.audio_track = MediaBlackhole()
        self.video_track = MediaBlackhole()
```

`MediaBlackhole()` is a sink that discards media - it doesn't forward or process it.

### Commented-Out Video Processing

The codebase includes commented-out video track classes:

```67:72:python/go2_webrtc/go2_connection.py
class Go2AudioTrack(AudioStreamTrack):
    kind = "audio"


class Go2VideoTrack(VideoStreamTrack):
    kind = "video"
```

These would need to be implemented if you want Python-based video processing.

### Video from Robot

```108:115:python/go2_webrtc/go2_connection.py
    def on_track(self, track):
        logger.debug("Receiving %s", track.kind)
        if track.kind == "audio":
            pass
            # self.audio_track.addTrack(track)
        elif track.kind == "video":
            # self.video_track.addTrack(track)
            pass
```

Currently, tracks are received but not processed by the Python backend.

## Video Codec and Encoding

The video stream uses standard WebRTC codecs:
- **H.264** or **VP8/VP9** for video compression
- **Opus** or **G.711** for audio

The specific codec is negotiated during the SDP offer/answer exchange based on both endpoints' capabilities.

## Three.js Video Visualization

The repository includes a Three.js-based visualization system (see `javascript/threejs.html`, `threejs.js`, `threejs.init.js`), but this appears to be for displaying 3D data (likely LIDAR voxel maps), not the robot's camera feed.

## Video Element Integration

The video is typically displayed in an HTML `<video>` element:

```html
<video id="video-frame" autoplay playsinline></video>
```

The key attributes:
- `autoplay`: Start playing automatically when stream arrives
- `playsinline`: Play inline on mobile (required for iOS)

## Video Stream Lifecycle

1. **Connection Established**: WebRTC handshake completes
2. **Tracks Arrive**: Robot sends video/audio tracks
3. **Track Event Fired**: Browser calls `trackEventHandler`
4. **Video Requested**: Client sends video "on" message
5. **Robot Starts Streaming**: Robot begins sending video frames
6. **Browser Displays**: Video appears in `<video>` element
7. **Ongoing**: Video continues streaming while connection is active

## Video Quality and Performance

### Resolution
- Typically: **640x480** or **1280x720**
- Set by robot's camera capabilities

### Frame Rate
- Usually: **30 FPS**
- May drop under network congestion

### Bandwidth
- Depends on resolution and quality settings
- Typical: **1-5 Mbps** for decent quality

## Troubleshooting Video Issues

### No Video Appears

**Checklist:**
1. Is connection validated? (Check for "Validation OK" message)
2. Was video "on" command sent?
3. Is video element present in DOM?
4. Check browser console for errors

### Video Stuttering

**Possible causes:**
- Insufficient network bandwidth
- High latency
- CPU overload on robot or browser
- WebRTC codec mismatch

### Video Quality Issues

**Solutions:**
- Check network speed
- Verify robot's camera isn't blocked/dirty
- Try adjusting robot's camera settings (if configurable)

## Audio Streaming

Audio is set up with `sendrecv` direction, meaning both directions:
- **Receive**: Robot can send audio (barks, status sounds, etc.)
- **Send**: Browser can send audio (voice commands, if implemented)

Currently, audio is not actively used in the provided examples, but the infrastructure is in place.

## Python Example (if implemented)

If you wanted to process video in Python:

```python
class Go2VideoTrack(VideoStreamTrack):
    kind = "video"
    
    def __init__(self):
        super().__init__()
        self.frames = queue.Queue()
    
    async def recv(self):
        # Get frame from robot
        frame = await self.frames.get()
        # Process frame (e.g., object detection)
        return frame

# In Go2Connection
self.pc.on("track", self.on_track)

def on_track(self, track):
    if track.kind == "video":
        self.video_track.addTrack(track)
```

## Advanced: Video Processing

Potential enhancements:

1. **Frame Processing**: Modify frames before display
2. **Recording**: Save stream to file
3. **Object Detection**: Analyze video for objects
4. **Streaming to Server**: Broadcast to multiple clients

Note: These would require implementing the commented-out video track classes.

## Next Steps

- Read [LIDAR Data Decoding](./07-lidar-decoding.md) for sensor data handling
- Check [Code Walkthrough](./08-code-walkthrough.md) for implementation details
- See [System Architecture](./01-architecture-overview.md) for the big picture

