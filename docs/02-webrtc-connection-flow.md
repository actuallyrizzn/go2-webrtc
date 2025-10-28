# WebRTC Connection Flow

This document explains the detailed flow of establishing a WebRTC connection with the Unitree GO2 robot.

## What is WebRTC?

WebRTC (Web Real-Time Communication) is a protocol that enables peer-to-peer communication between browsers and other devices. For this project, it provides:
- **Low-latency video streaming** from the robot
- **Bidirectional data channels** for commands and sensor data
- **Direct peer-to-peer** connection (after initial signaling)

## Connection Flow Diagram

```
Browser                          Server                     Python Backend                 Robot
  │                                │                             │                           │
  │  1. Create RTCPeerConnection   │                             │                           │
  │  2. Add transceivers           │                             │                           │
  │  3. Create offer               │                             │                           │
  │  4. Set local description      │                             │                           │
  │─────────────────────────────────┼─────────────────────────────┼───────────────────────────│
  │  5. POST /offer (with SDP)     │                             │                           │
  │────────────────────────────────▶│                             │                           │
  │                                 │  Convert SDP to object      │                           │
  │                                 │─────────────────────────────▶│                           │
  │                                 │                             │ Get robot's public key    │
  │                                 │                             │───────────────────────────▶│
  │                                 │                             │                           │ GET /con_notify
  │                                 │                             │◀──────────────────────────│
  │                                 │                             │ (Returns RSA public key)  │
  │                                 │                             │                           │
  │                                 │                             │ Encrypt SDP with AES     │
  │                                 │                             │ Encrypt AES key with RSA │
  │                                 │                             │                           │
  │                                 │                             │ POST to /con_ing_XXXX    │
  │                                 │                             │───────────────────────────▶│
  │                                 │                             │                           │
  │                                 │                             │◀──── (Encrypted SDP) ─────│
  │                                 │                             │                           │
  │                                 │                             │ Decrypt response          │
  │                                 │◀─────────────────────────────│                           │
  │◀────────────────────────────────│ Return answer SDP           │                           │
  │                                 │                             │                           │
  │  6. Set remote description      │                             │                           │
  │  7. ICE candidates exchanged    │                             │                           │
  │  8. WebRTC connection open       │                             │                           │
  │──────────────────────────────────────────────────────────────────────────────────────────▶│
  │  9. WebRTC DataChannel open     │                             │                           │
```

## Step-by-Step Breakdown

### Step 1: Browser Initialization

When you click "Connect" in the browser, the JavaScript code initializes a WebRTC peer connection:

```71:83:javascript/go2webrtc.js
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
```

**What happens:**
1. `createOffer()` - Generates an SDP offer containing the browser's capabilities and media requirements
2. `setLocalDescription(offer)` - Stores the offer locally
3. The offer is sent to the signaling server

### Step 2: Offer Transmission to Server

```85:124:javascript/go2webrtc.js
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

**Key components of the offer:**
- `token`: Authentication token for the robot
- `sdp`: Session Description Protocol (the actual offer)
- `id`: Network identifier ("STA_localNetwork")
- `ip`: Robot's IP address

### Step 3: Server Processing

```62:88:javascript/server.py
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

The server:
1. Receives the JSON offer
2. Converts it to an `RTCSessionDescription` object
3. Passes it to Python backend's `get_peer_answer()` method
4. Returns the answer to the browser

### Step 4: Python Backend - Encrypted Handshake with Robot

This is where the cryptographic magic happens! The Python backend must exchange encrypted messages with the robot.

#### 4a. Getting the Robot's Public Key

```234:248:python/go2_webrtc/go2_connection.py
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
```

The robot responds with a base64-encoded JSON containing an RSA public key:
```json
{
  "data1": "-----BEGIN PUBLIC KEY-----\n<key material>\n-----END PUBLIC KEY-----"
}
```

#### 4b. Encrypting the SDP Offer

```248:270:python/go2_webrtc/go2_connection.py
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
```

**Why this double encryption?**
- **AES** is fast for encrypting large data (the SDP offer)
- **RSA** is used for key exchange (small data - just the AES key)
- This hybrid approach is common in secure systems

#### 4c. Sending Encrypted Data to Robot

```273:290:python/go2_webrtc/go2_connection.py
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
```

The robot decrypts, processes the offer, generates an answer, encrypts it, and sends it back.

### Step 5: Browser Receives Answer

Back in the browser:

```108:119:javascript/go2webrtc.js
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
```

The browser:
1. Sets the remote description (the robot's answer)
2. ICE candidates are exchanged (for NAT traversal)
3. WebRTC connection is established
4. Heartbeat timer starts (keeps connection alive)

## ICE and NAT Traversal

After the SDP exchange, WebRTC uses ICE (Interactive Connectivity Establishment) to establish the actual peer-to-peer connection:

- **STUN servers**: Help discover your public IP
- **TURN servers**: Relay traffic if direct connection fails
- **ICE candidates**: Network paths that could work

The browser and robot exchange ICE candidates and negotiate which path to use.

## Data Channel Establishment

Parallel to the media connection, a data channel is set up:

```19:19:javascript/go2webrtc.js
    this.channel = this.pc.createDataChannel("data");
```

This channel enables:
- Command sending
- State information
- Validation messages
- Heartbeat messages

## Connection States

The WebRTC connection goes through these states:

1. **"new"** - Initial state
2. **"connecting"** - ICE negotiation in progress
3. **"connected"** - Peer-to-peer connection established
4. **"disconnected"** - Temporary network issue
5. **"failed"** - Connection failed
6. **"closed"** - Connection closed

## Common Issues

### Connection Fails at Step 4

**Symptoms:** Timeout or 401 error when reaching robot

**Possible causes:**
- Incorrect IP address
- Robot not on same network
- Invalid authentication token
- Robot firewall blocking

### SDP Exchange Succeeds but No Connection

**Symptoms:** "WebRTC connection established" but no data

**Possible causes:**
- NAT traversal failed (need TURN server)
- Robot not responding on WebRTC ports
- Network configuration issues

## Next Steps

- Learn about [Security & Authentication](./03-security-authentication.md) to understand the encryption details
- Read [Data Channels & Messaging](./04-data-channels-messaging.md) to understand the command protocol

