# System Architecture Overview

## What is This Project?

The `go2-webrtc` project provides a WebRTC-based API for controlling Unitree GO2 robots. It enables real-time, low-latency communication between a web client and the robot, allowing you to:
- Stream video from the robot
- Send commands to control movement and actions
- Receive sensor data (LIDAR, state information)
- Communicate bidirectionally over encrypted channels

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      BROWSER/CLIENT                         │
│                                                              │
│  ┌──────────────┐     ┌──────────────┐   ┌──────────────┐ │
│  │  index.js    │────▶│ go2webrtc.js │──▶│   utils.js   │ │
│  │  (UI Logic)  │     │ (WebRTC Core)│   │ (Encryption) │ │
│  └──────────────┘     └──────────────┘   └──────────────┘ │
│         │                                                    │
│         │ HTTP POST /offer                                  │
└─────────┼────────────────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│                 SERVER (server.py)                           │
│                                                              │
│  Simple HTTP server that proxies WebRTC offer to Python    │
│  - CORS handling                                             │
│  - Converts JSON SDP to RTCSessionDescription              │
│  - Forwards to Python backend                               │
└──────────────────┬──────────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────────┐
│              PYTHON BACKEND (go2_connection.py)             │
│                                                              │
│  ┌────────────────┐    ┌──────────────────┐                │
│  │ WebRTC Process │───▶│ Authentication   │                │
│  │ (aiortc)       │    │ (RSA/AES Crypto) │                │
│  └────────────────┘    └──────────────────┘                │
│         │                                                      │
│         ▼                                                      │
│  ┌──────────────────┐  ┌────────────────────────┐           │
│  │  Data Channels   │  │  Media Streams        │           │
│  │  (Commands/Data) │  │  (Video/Audio)        │           │
│  └──────────────────┘  └────────────────────────┘           │
└───────────┬──────────────────────────────────────────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│                    UNITREE GO2 ROBOT                         │
│                                                              │
│  Port 8081: Initial WebRTC signaling                         │
│  Port 9991: Encrypted connection endpoint                   │
│  - RSA public key exchange                                   │
│  - AES encrypted SDP handshake                               │
│  - WebRTC peer connection                                    │
└─────────────────────────────────────────────────────────────┘
```

## Component Breakdown

### 1. JavaScript Client (`javascript/`)

**Files:**
- `go2webrtc.js` - Core WebRTC connection class
- `index.js` - UI event handlers and gamepad/keyboard controls
- `utils.js` - Cryptographic utilities (MD5, Base64)
- `constants.js` - Command IDs and data channel type definitions

**Responsibilities:**
- Creates WebRTC peer connection with robot
- Handles offer/answer SDP exchange
- Manages data channel for commands and state
- Receives video streams
- User interface for control

### 2. Python Backend (`python/go2_webrtc/`)

**Files:**
- `go2_connection.py` - Main connection logic, encryption, WebRTC handling
- `constants.py` - Command definitions matching JavaScript
- `lidar_decoder.py` - WASM-based decoder for compressed LIDAR data
- `go2_cv_video.py` - Video processing (if used)

**Responsibilities:**
- Server-side WebRTC peer connection management
- RSA/AES encryption for secure connection
- Robot authentication and validation
- Data channel message handling
- LIDAR data decompression

### 3. HTTP Signaling Server (`javascript/server.py`)

**Responsibilities:**
- Accepts WebRTC offers from browser
- Forwards offers to Python backend
- Returns answers from Python to browser
- Handles CORS for web clients

**Note:** This is essentially a minimal proxy that bridges the JavaScript WebRTC API with the Python backend.

## Key Technologies

### WebRTC
- **Why WebRTC?** Low-latency, peer-to-peer media streaming
- **Handshake:** SDP (Session Description Protocol) offer/answer exchange
- **Channels:** RTCPeerConnection for media, DataChannel for messages

### Encryption
- **RSA:** For secure key exchange (robot's public key)
- **AES:** For encrypting SDP data during handshake
- **Token-based Auth:** MD5 hashed tokens for ongoing validation

### Communication Protocol
- **Data Channels:** Bidirectional JSON messaging
- **Topics:** MQTT-like topic system (`rt/api/sport/request`, etc.)
- **Message Types:** validation, heartbeat, request, response, etc.

## Data Flow Examples

### Connection Establishment Flow

1. Browser creates WebRTC offer
2. Browser sends offer to `http://localhost:8081/offer`
3. Server forwards to Python backend
4. Python backend:
   - Fetches robot's public key from `http://robot_ip:9991/con_notify`
   - Encrypts SDP with AES
   - Encrypts AES key with robot's RSA public key
   - Sends encrypted payload to `http://robot_ip:9991/con_ing_XXXX`
5. Robot decrypts and responds with answer
6. Python returns answer to browser
7. Browser establishes WebRTC connection

### Command Flow

1. User presses key or uses joystick
2. JavaScript constructs command JSON:
   ```json
   {
     "type": "msg",
     "topic": "rt/api/sport/request",
     "data": {
       "header": {"identity": {"id": 12345, "api_id": 1008}},
       "parameter": "{\"x\": 0.8, \"y\": 0, \"z\": 0}"
     }
   }
   ```
3. Command sent via DataChannel
4. Robot receives command, executes action
5. Robot may send response back via DataChannel

### Video Streaming Flow

1. After validation, browser sends video on request:
   ```json
   {"type": "vid", "topic": "", "data": "on"}
   ```
2. Robot starts sending video stream via WebRTC media track
3. Browser receives video track events
4. Video displayed in `<video>` element

## Key Concepts

### Dual Communication Channels

The system uses two parallel channels:

1. **WebRTC Peer Connection** - For media (video/audio streams)
2. **WebRTC Data Channel** - For structured data (commands, state, sensor data)

This separation allows low-latency video streaming while maintaining reliable command delivery.

### State Management

- **Connection State:** Tracks WebRTC connection lifecycle
- **Validation State:** Tracks authentication status
- **Message Callbacks:** Map of registered callbacks waiting for responses

### Heartbeat Mechanism

Every 2 seconds, the client sends a heartbeat message to keep the connection alive:
```javascript
{
  "type": "heartbeat",
  "data": {
    "timeInStr": "2024-01-01 12:00:00",
    "timeInNum": 1704110400
  }
}
```

## Next Steps

- Read about the detailed [WebRTC Connection Flow](./02-webrtc-connection-flow.md)
- Understand [Security & Authentication](./03-security-authentication.md)
- Learn about [Data Channels & Messaging](./04-data-channels-messaging.md)

