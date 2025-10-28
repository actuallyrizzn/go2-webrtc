# Code Walkthrough

This document provides a detailed walkthrough of the key implementation files, explaining how they work and how they fit together.

## Table of Contents

1. [JavaScript Client Files](#javascript-client-files)
2. [Python Backend Files](#python-backend-files)
3. [Integration Flow](#integration-flow)
4. [Key Algorithms](#key-algorithms)

## JavaScript Client Files

### go2webrtc.js - The Core WebRTC Class

This is the heart of the JavaScript client implementation.

#### Class Initialization

```10:27:javascript/go2webrtc.js
export class Go2WebRTC {
  constructor(token, robotIP, messageCallback) {
    this.token = token;
    this.robotIP = robotIP;
    this.messageCallback = messageCallback;

    this.msgCallbacks = new Map();
    this.validationResult = "PENDING";
    this.pc = new RTCPeerConnection({ sdpSemantics: "unified-plan" });
    this.channel = this.pc.createDataChannel("data");

    this.pc.addTransceiver("video", { direction: "recvonly" });
    this.pc.addTransceiver("audio", { direction: "sendrecv" });
    this.pc.addEventListener("track", this.trackEventHandler.bind(this));
    this.channel.onmessage = this.messageEventHandler.bind(this);

    this.heartbeatTimer = null;
  }
```

**Key components:**
- `token`: Authentication token for robot
- `robotIP`: Robot's IP address
- `messageCallback`: Optional callback for incoming messages
- `msgCallbacks`: Map of pending responses
- `pc`: WebRTC peer connection
- `channel`: Data channel for structured messages

**WebRTC Setup:**
- `createDataChannel("data")`: Creates bidirectional data channel
- `addTransceiver("video")`: Sets up video reception
- `addTransceiver("audio")`: Sets up bidirectional audio

#### Offer Creation and Signaling

```71:124:javascript/go2webrtc.js
  initSDP() {
    this.pc
      .createOffer()
      .then((offer) => this.pc.setLocalDescription(offer))
      .then(() => {
        console.log("Offer created");
        logMessage("Offer created");
        console.log(this.pc.localDescription);
        logMessage(this.pc.localDescription);
        this.initSignaling();
      })
      .catch(console.error);
  }

  initSignaling() {
    var answer = {
      token: this.token,
      id: "STA_localNetwork",
      type: "offer",
      ip: this.robotIP
    };
    answer["sdp"] = this.pc.localDescription.sdp;
    console.log(answer);

    const options = {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
      },
      body: JSON.stringify(answer),
    };

    fetch(`http://${window.location.hostname}:8081/offer`, options)
      .then((response) => {
        console.log(`statusCode: ${response.status}`);
        return response.json();
      })
      .then((data) => {
        console.log("Response from signaling server:" + JSON.stringify(data));
        logMessage("Establishing connection...");
        this.pc
          .setRemoteDescription(data)
          .then(() => {
            logMessage("WebRTC connection established");
            this.startHeartbeat();
          })
          .catch((e) => {
            console.log(e);
          });
      })
      .catch((error) => {
        console.error("Error sending message:", error);
      });
  }
```

**Flow:**
1. Create WebRTC offer (SDP description)
2. Set as local description
3. Package offer with token/IP
4. POST to signaling server
5. Receive answer
6. Set as remote description
7. Start heartbeat timer

#### Message Publishing

```187:214:javascript/go2webrtc.js
  publish(topic, data, channelType) {
    logMessage(
      `<- msg type:${channelType} topic:${topic} data:${JSON.stringify(data)}`
    );
    return new Promise((resolve, reject) => {
      if (this.channel && this.channel.readyState === "open") {
        const msg = {
          type: channelType || DataChannelType.MSG,
          topic: topic,
          data: data,
        };
        this.channel.send(JSON.stringify(msg));
        const id =
          data && data.uuid
            ? data.uuid
            : data && data.header && data.header.identity.id;
        this.saveResolve(
          channelType || DataChannelType.MSG,
          topic,
          resolve,
          id
        );
      } else {
        console.error("data channel is not open", topic);
        reject("data channel is not open");
      }
    });
  }
```

**Features:**
- Validates data channel is open
- Constructs message with type/topic/data
- Sends as JSON string
- Stores promise for response matching
- Returns promise for async handling

### index.js - UI and Event Handlers

#### Connection Handling

```43:59:javascript/index.js
// Function to handle connect button click
function handleConnectClick() {
  // You can add connection logic here
  // For now, let's just log the values
  const token = document.getElementById("token").value;
  const robotIP = document.getElementById("robot-ip").value;

  console.log("Token:", token);
  console.log("Robot IP:", robotIP);
  logMessage(`Connecting to robot on ip ${robotIP}...`);

  // Save the values to localStorage
  saveValuesToLocalStorage();

  // Initialize RTC
  globalThis.rtc = new Go2WebRTC(token, robotIP);
  globalThis.rtc.initSDP();
}
```

**Process:**
1. Read token and IP from form
2. Log to console for debugging
3. Save to localStorage for persistence
4. Create Go2WebRTC instance
5. Initialize SDP offer

#### Control Input Handling

```100:131:javascript/index.js
function joystickTick(joyLeft, joyRight) {
  let x,y,z = 0;
  let gpToUse = document.getElementById("gamepad").value;
  if (gpToUse !== "NO") {
    const gamepads = navigator.getGamepads();
    let gp = gamepads[gpToUse];
    
    // LB must be pressed
    if (gp.buttons[4].pressed == true) {
      x = -1 * applyGamePadDeadzeone(gp.axes[1], 0.25);
      y = -1 * applyGamePadDeadzeone(gp.axes[2], 0.25);
      z = -1 * applyGamePadDeadzeone(gp.axes[0], 0.25);
    } 
  } else {
     y = -1 * (joyRight.GetPosX() - 100) / 50;
     x = -1 * (joyLeft.GetPosY() - 100) / 50;
     z = -1 * (joyLeft.GetPosX() - 100) / 50;
  }

  if (x === 0 && y === 0 && z === 0) {
    return;
  }

  if (x == undefined || y == undefined || z == undefined) {
    return;
  }

  console.log("Joystick Linear:", x, y, z);

  if(globalThis.rtc == undefined) return;
  globalThis.rtc.publishApi("rt/api/sport/request", 1008, JSON.stringify({x: x, y: y, z: z}));
}
```

**Input sources:**
- Physical gamepads (Xbox controllers)
- On-screen joysticks
- Keyboard

**Deadzone handling:**
```96:98:javascript/index.js
function applyGamePadDeadzeone(value, th) {
  return Math.abs(value) > th ? value : 0
}
```

Small controller drift is ignored.

### utils.js - Cryptographic Utilities

```1:20:javascript/utils.js
// import { MD5} from './md5.js';


function hexToBase64(r) {
  var o;
  const n =
    (o = r.match(/.{1,2}/g)) == null ? void 0 : o.map((s) => parseInt(s, 16));
  return window.btoa(String.fromCharCode.apply(null, n));
}

export const encryptKey = (r) => {
  const n = `UnitreeGo2_${r}`,
    o = encryptByMd5(n);
  return hexToBase64(o);
};

function encryptByMd5(r) {
  return md5(r).toString();
}
```

**Validation encryption:**
1. Prepend `"UnitreeGo2_"` to challenge
2. Compute MD5 hash
3. Convert hex to Base64

## Python Backend Files

### go2_connection.py - Main Backend

This file handles the server-side WebRTC connection and encryption.

#### Connection Initialization

```75:104:python/go2_webrtc/go2_connection.py
class Go2Connection:
    def __init__(
        self, ip=None, token="", on_validated=None, on_message=None, on_open=None
    ):
        self.pc = RTCPeerConnection()
        self.ip = ip
        self.token = token
        self.validation_result = "PENDING"
        self.on_validated = on_validated
        self.on_message = on_message
        self.on_open = on_open

        # self.audio_track = Go2AudioTrack()
        # self.video_track = Go2VideoTrack()
        # self.video_track = Go2CvVideo()
        self.audio_track = MediaBlackhole()
        self.video_track = MediaBlackhole()

        # Create and add a data channel
        self.data_channel = self.pc.createDataChannel("data", id=2, negotiated=False)
        self.data_channel.on("open", self.on_data_channel_open)
        self.data_channel.on("message", self.on_data_channel_message)

        # self.pc.addTransceiver("video", direction="recvonly")
        # self.pc.addTransceiver("audio", direction="sendrecv")
        # self.pc.addTrack(AudioStreamTrack())

        self.pc.on("track", self.on_track)
        self.pc.on("connectionstatechange", self.on_connection_state_change)
```

**Key differences from JS:**
- Uses `aiortc` library (Python WebRTC)
- `MediaBlackhole()` discards media (not forwarding)
- Event handlers use `.on()` instead of `.addEventListener()`

#### Encrypted Handshake

```234:296:python/go2_webrtc/go2_connection.py
    @staticmethod
    def get_peer_answer(sdp_offer, token, robot_ip):
        sdp_offer_json = {
            "id": "STA_localNetwork",
            "sdp": sdp_offer.sdp,
            "type": sdp_offer.type,
            "token": token,
        }

        new_sdp = json.dumps(sdp_offer_json)
        url = f"http://{robot_ip}:9991/con_notify"
        response = Go2Connection.make_local_request(url, body=None, headers=None)

        if response:
            # Decode the response text from base64
            decoded_response = base64.b64decode(response.text).decode("utf-8")

            # Parse the decoded response as JSON
            decoded_json = json.loads(decoded_response)

            # Extract the 'data1' field from the JSON
            data1 = decoded_json.get("data1")

            # Extract the public key from 'data1'
            public_key_pem = data1[10 : len(data1) - 10]
            path_ending = Go2Connection.calc_local_path_ending(data1)

            # Generate AES key
            aes_key = Go2Connection.generate_aes_key()

            # Load Public Key
            public_key = Go2Connection.rsa_load_public_key(public_key_pem)

            # Encrypt the SDP and AES key
            body = {
                "data1": Go2Connection.aes_encrypt(new_sdp, aes_key),
                "data2": Go2Connection.rsa_encrypt(aes_key, public_key),
            }

            # URL for the second request
            url = f"http://{robot_ip}:9991/con_ing_{path_ending}"

            # Set the appropriate headers for URL-encoded form data
            headers = {"Content-Type": "application/x-www-form-urlencoded"}

            # Send the encrypted data via POST
            response = Go2Connection.make_local_request(
                url, body=json.dumps(body), headers=headers
            )

            # If response is successful, decrypt it
            if response:
                decrypted_response = Go2Connection.aes_decrypt(response.text, aes_key)
                peer_answer = json.loads(decrypted_response)

                return peer_answer
            else:
                raise ValueError(f"Failed to get answer from server")

        else:
            raise ValueError(
                "Failed to receive initial public key response with new method."
            )
```

**Process:**
1. Package SDP offer
2. GET robot's public key from `/con_notify`
3. Parse base64 response
4. Extract public key
5. Generate random AES key
6. Encrypt SDP with AES
7. Encrypt AES key with RSA
8. POST encrypted data to `/con_ing_XXXX`
9. Receive encrypted answer
10. Decrypt response
11. Return SDP answer

#### Crypto Implementation

**AES Encryption:**
```400:410:python/go2_webrtc/go2_connection.py
    @staticmethod
    def aes_encrypt(data: str, key: str) -> str:
        """Encrypt the given data using AES (ECB mode with PKCS5 padding)."""
        # Ensure key is 32 bytes for AES-256
        key_bytes = key.encode("utf-8")
        # Pad the data to ensure it is a multiple of block size
        padded_data = Go2Connection.pad(data)
        # Create AES cipher in ECB mode
        cipher = AES.new(key_bytes, AES.MODE_ECB)
        encrypted_data = cipher.encrypt(padded_data)
        encoded_encrypted_data = base64.b64encode(encrypted_data).decode("utf-8")
        return encoded_encrypted_data
```

**RSA Encryption:**
```413:426:python/go2_webrtc/go2_connection.py
    @staticmethod
    def rsa_encrypt(data: str, public_key: RSA.RsaKey) -> str:
        """Encrypt data using RSA and a given public key."""
        cipher = PKCS1_v1_5.new(public_key)
        # Maximum chunk size for encryption with RSA/ECB/PKCS1Padding is key size - 11 bytes
        max_chunk_size = public_key.size_in_bytes() - 11
        data_bytes = data.encode("utf-8")
        encrypted_bytes = bytearray()
        for i in range(0, len(data_bytes), max_chunk_size):
            chunk = data_bytes[i : i + max_chunk_size]
            encrypted_chunk = cipher.encrypt(chunk)
            encrypted_bytes.extend(encrypted_chunk)
        # Base64 encode the final encrypted data
        encoded_encrypted_data = base64.b64encode(encrypted_bytes).decode("utf-8")
        return encoded_encrypted_data
```

### server.py - HTTP Signalling Server

```53:88:javascript/server.py
class CORSRequestHandler(http.server.SimpleHTTPRequestHandler):
    def do_OPTIONS(self):
        # Handle CORS preflight request
        self.send_response(200, "ok")
        self.send_header("Access-Control-Allow-Origin", "*")
        self.send_header("Access-Control-Allow-Methods", "POST, OPTIONS")
        self.send_header("Access-Control-Allow-Headers", "Content-Type")
        self.end_headers()

    def do_POST(self):
        if self.path == "/offer":
            # Read the length of the data
            content_length = int(self.headers["Content-Length"])
            # Read the incoming data
            post_data = self.rfile.read(content_length)
            # Parse the JSON data
            try:
                data = SDPDict(json.loads(post_data))
            except json.JSONDecodeError:
                self.send_response(400)
                self.end_headers()
                return

            # Convert the dictionary to RTCSessionDescription
            sdp_offer = RTCSessionDescription(sdp=data.sdp, type=data.type)
            
            response_data = go2_webrtc.Go2Connection.get_peer_answer(
                sdp_offer, data.token, data.ip
            )
            self.send_response(200)
            self.send_header("Content-Type", "application/json")
            self.send_header("Access-Control-Allow-Origin", "*")
            self.end_headers()

            self.wfile.write(json.dumps(response_data).encode("utf-8"))
```

**Role:** Simple HTTP proxy between browser and Python backend.

**Features:**
- CORS handling for web clients
- Converts JSON to `RTCSessionDescription`
- Calls Python backend
- Returns response to browser

### constants.py - Command Definitions

Both JavaScript and Python share the same command IDs:

```24:61:python/go2_webrtc/constants.py
SPORT_CMD = {
    1001: "Damp",
    1002: "BalanceStand",
    1003: "StopMove",
    1004: "StandUp",
    # ... (see full file)
}
```

This ensures **compatibility** between JavaScript and Python implementations.

## Integration Flow

### Complete Flow Example

1. **User clicks "Connect"** (`index.js` line 57)
   - Reads token/IP from form
   - Creates `Go2WebRTC` instance

2. **Offer created** (`go2webrtc.js` line 72)
   - `createOffer()` generates SDP

3. **Sent to server** (`go2webrtc.js` line 103)
   - POST to `http://localhost:8081/offer`

4. **Server forwards** (`server.py` line 79)
   - Calls `go2_webrtc.Go2Connection.get_peer_answer()`

5. **Encrypted handshake** (`go2_connection.py` line 235)
   - Fetches robot's public key
   - Encrypts with AES + RSA
   - Sends to robot

6. **Robot responds** (`go2_connection.py` line 286)
   - Decrypts answer
   - Returns to server

7. **Browser receives** (`go2webrtc.js` line 111)
   - Sets remote description
   - Connection established!

8. **Validation** (`go2webrtc.js` line 143)
   - Robot sends challenge
   - Client encrypts response
   - Robot validates

9. **Ready to use** (`go2webrtc.js` line 115)
   - Heartbeat starts
   - Video can be enabled
   - Commands can be sent

## Key Algorithms

### Path Endpoint Calculation

```350:377:python/go2_webrtc/go2_connection.py
    @staticmethod
    def calc_local_path_ending(data1):
        # Initialize an array of strings
        strArr = ["A", "B", "C", "D", "E", "F", "G", "H", "I", "J"]

        # Extract the last 10 characters of data1
        last_10_chars = data1[-10:]

        # Split the last 10 characters into chunks of size 2
        chunked = [last_10_chars[i : i + 2] for i in range(0, len(last_10_chars), 2)]

        # Initialize an empty list to store indices
        arrayList = []

        # Iterate over the chunks and find the index of the second character in strArr
        for chunk in chunked:
            if len(chunk) > 1:
                second_char = chunk[1]
                try:
                    index = strArr.index(second_char)
                    arrayList.append(index)
                except ValueError:
                    # Handle case where the character is not found in strArr
                    print(f"Character {second_char} not found in strArr.")

        # Convert arrayList to a string without separators
        joinToString = "".join(map(str, arrayList))

        return joinToString
```

**Purpose:** Generates dynamic endpoint path for encrypted communication.

**Algorithm:**
1. Take last 10 chars of robot's response
2. Split into pairs
3. For each pair, find index of 2nd char in ["A".."J"]
4. Join indices into string
5. Use as path suffix: `/con_ing_{result}`

**Example:**
```
data1 = "...AB1CD2EF3GH4IJ5"
last_10 = "AB1CD2EF3GH"
pairs = ["AB", "1C", "D2", "EF", "3G", "H"]
second_chars = ["B", "C", "2", "F", "G", "H"] (not in strArr - ignoring 2)
indices = [1, 2, 5, 7, 8]  # B=1, C=2, F=5, G=7, H=8
path = "12578"
URL = "http://robot/con_ing_12578"
```

### Unique ID Generation

```324:327:python/go2_webrtc/go2_connection.py
    @staticmethod
    def generate_id():
        return int(
            datetime.datetime.now().timestamp() * 1000 % 2147483648
        ) + random.randint(0, 999)
```

**Formula:** `(timestamp_ms % 2^31) + random(0-999)`

**Why:** Ensures unique, sortable IDs within 32-bit signed int range.

## Summary

The codebase demonstrates a **well-architected WebRTC system** with:

- **Clean separation**: JS for UI, Python for backend, HTTP for signaling
- **Strong security**: Hybrid RSA/AES encryption
- **Bidirectional**: Commands down, video/state up
- **Robust**: Error handling, heartbeats, validation
- **Extensible**: Modular design allows easy additions

## Next Steps

- Dive into specific files for more detail
- Experiment with adding features
- Read [System Architecture](./01-architecture-overview.md) for big picture
- Check other docs for specialized topics

