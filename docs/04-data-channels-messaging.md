# Data Channels & Messaging

This document explains the data channel communication protocol used to send commands and receive data from the robot.

## What is a Data Channel?

A WebRTC Data Channel is a bidirectional communication pathway that carries arbitrary data between peers. Think of it like a WebSocket, but built into WebRTC.

In this project, the data channel carries:
- **Commands** (move, sit, stand up, etc.)
- **State information** (robot status, sensor data)
- **Validation messages**
- **Heartbeat messages**
- **LIDAR data** (compressed voxel maps)

## Message Structure

All messages follow this JSON structure:

```json
{
  "type": "msg",           // Message type
  "topic": "rt/api/sport/request",  // MQTT-like topic
  "data": {                 // Actual payload
    "header": {
      "identity": {
        "id": 12345,        // Unique message ID
        "api_id": 1008      // Command ID
      }
    },
    "parameter": "{\"x\": 0.8, \"y\": 0, \"z\": 0}"  // Command data (JSON string)
  }
}
```

## Message Types

### Defined Types

```42:113:javascript/constants.js
export const DataChannelType = {};

(function initializeDataChannelTypes(types) {
  const defineType = (r, name) => (r[name.toUpperCase()] = name.toLowerCase());

  defineType(types, "VALIDATION");
  defineType(types, "SUBSCRIBE");
  defineType(types, "UNSUBSCRIBE");
  defineType(types, "MSG");
  defineType(types, "REQUEST");
  defineType(types, "RESPONSE");
  defineType(types, "VID");
  defineType(types, "AUD");
  defineType(types, "ERR");
  defineType(types, "HEARTBEAT");
  defineType(types, "RTC_INNER_REQ");
  defineType(types, "RTC_REPORT");
  defineType(types, "ADD_ERROR");
  defineType(types, "RM_ERROR");
  defineType(types, "ERRORS");
})(DataChannelType);
```

### Type Descriptions

| Type | Purpose | Direction |
|------|---------|-----------|
| `validation` | Authentication challenge/response | Bidirectional |
| `heartbeat` | Keep-alive messages | Browser → Robot |
| `msg` | Standard commands | Browser → Robot |
| `request` | API requests with responses | Bidirectional |
| `response` | Responses to requests | Robot → Browser |
| `vid` | Video stream control | Browser → Robot |
| `err` | Error messages | Robot → Browser |

## Core Message Handling

### Sending Messages

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

**Key points:**
1. Checks if data channel is open
2. Constructs message with type, topic, and data
3. Stringifies and sends
4. Saves promise for response tracking

### Receiving Messages

```37:60:javascript/go2webrtc.js
  messageEventHandler(event) {
    if (
      event.data &&
      event.data.includes &&
      !event.data.includes("heartbeat")
    ) {
      console.log("onmessage", event);
      this.handleDataChannelMessage(event);
    }
  }

  handleDataChannelMessage(event) {
    const data =
      typeof event.data == "string"
        ? JSON.parse(event.data)
        : this.dealArrayBuffer(event.data);
    if (data.type === DataChannelType.VALIDATION) {
      this.rtcValidation(data);
    }

    if (this.messageCallback) {
      this.messageCallback(data);
    }
  }
```

Messages come in two formats:
1. **String (JSON)** - Standard messages
2. **ArrayBuffer** - Binary data (LIDAR, sensor data)

## Common Message Patterns

### 1. Movement Commands

**Example: Move forward**

```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1008, JSON.stringify({
  x: 0.8,  // Forward velocity
  y: 0,    // Sideways velocity
  z: 0     // Rotational velocity
}));
```

**Implementation:**

```216:226:javascript/go2webrtc.js
  publishApi(topic, api_id, data) {
    const uniqID =
      (new Date().valueOf() % 2147483648) + Math.floor(Math.random() * 1e3);

    console.log("Command:", api_id);

    this.publish(topic, {
      header: { identity: { id: uniqID, api_id: api_id} },
      parameter: data
    });
  }
```

### 2. Simple Commands

**Example: Stand up**

```javascript
// Using pre-defined command
globalThis.rtc.publishApi(
  "rt/api/sport/request",
  SPORT_CMD["StandUp"],  // 1004
  JSON.stringify(1004)
);
```

### 3. Heartbeat Messages

Sent every 2 seconds to keep connection alive:

```126:141:javascript/go2webrtc.js
  startHeartbeat() {
    this.heartbeatTimer = window.setInterval(() => {
      const date = new Date();
      (this.channel == null ? void 0 : this.channel.readyState) === "open" &&
        (this.channel == null ||
          this.channel.send(
            JSON.stringify({
              type: DataChannelType.HEARTBEAT,
              data: {
                timeInStr: this.formatDate(date),
                timeInNum: Math.floor(date.valueOf() / 1e3),
              },
            })
          ));
    }, 2e3);
  }
```

### 4. Video Control

```javascript
// Turn video on
globalThis.rtc.publish("", "on", DataChannelType.VID);

// Turn video off
globalThis.rtc.publish("", "off", DataChannelType.VID);
```

## Binary Message Handling

Some data (like LIDAR) comes as binary ArrayBuffer:

```62:69:javascript/go2webrtc.js
  dealArrayBuffer(n) {
    const o = new Uint16Array(n.slice(0, 2)),
      s = n.slice(4, 4 + o[0]),
      c = n.slice(4 + o[0]),
      u = new TextDecoder("utf-8"),
      l = JSON.parse(u.decode(s));
    return (l.data.data = c), l;
  }
```

**Structure:**
```
[2 bytes: length][2 bytes: ?][JSON: metadata][Binary: data]
```

**Python equivalent:**

```330:347:python/go2_webrtc/go2_connection.py
    @staticmethod
    def deal_array_buffer(n):
        # Unpack the first 2 bytes as an unsigned short (16-bit) to get the length
        length = struct.unpack("H", n[:2])[0]

        # Extract the JSON segment and the remaining data
        json_segment = n[4 : 4 + length]
        remaining_data = n[4 + length :]

        # Decode the JSON segment from UTF-8 and parse it
        json_str = json_segment.decode("utf-8")
        obj = json.loads(json_str)

        decoded_data = decoder.decode(remaining_data, obj["data"])

        # Attach the remaining data to the object
        obj["data"]["data"] = decoded_data

        return obj
```

## Topic System (MQTT-like)

The system uses an MQTT-like topic system for message routing:

```120:153:python/go2_webrtc/constants.py
RTC_TOPIC = {
    "LOW_STATE": "rt/lf/lowstate",
    "MULTIPLE_STATE": "rt/multiplestate",
    "FRONT_PHOTO_REQ": "rt/api/videohub/request",
    "ULIDAR_SWITCH": "rt/utlidar/switch",
    "ULIDAR": "rt/utlidar/voxel_map",
    "ULIDAR_ARRAY": "rt/utlidar/voxel_map_compressed",
    "ULIDAR_STATE": "rt/utlidar/lidar_state",
    "ROBOTODOM": "rt/utlidar/robot_pose",
    "UWB_REQ": "rt/api/uwbswitch/request",
    "UWB_STATE": "rt/uwbstate",
    "LOW_CMD": "rt/lowcmd",
    "WIRELESS_CONTROLLER": "rt/wirelesscontroller",
    "SPORT_MOD": "rt/api/sport/request",
    "SPORT_MOD_STATE": "rt/sportmodestate",
    "LF_SPORT_MOD_STATE": "rt/lf/sportmodestate",
    "BASH_REQ": "rt/api/bashrunner/request",
    "SELF_TEST": "rt/selftest",
    "GRID_MAP": "rt/mapping/grid_map",
    "SERVICE_STATE": "rt/servicestate",
    "GPT_FEEDBACK": "rt/gptflowfeedback",
    "VUI": "rt/api/vui/request",
    "OBSTACLES_AVOID": "rt/api/obstacles_avoid/request",
```

### Common Topics

- `rt/api/sport/request` - Sport mode commands (movement, poses, etc.)
- `rt/lf/lowstate` - Low-level state feedback
- `rt/utlidar/voxel_map` - LIDAR voxel map data
- `rt/lowcmd` - Low-level commands

## Command IDs

Sport commands are defined by numeric IDs:

```24:61:python/go2_webrtc/constants.py
SPORT_CMD = {
    1001: "Damp",
    1002: "BalanceStand",
    1003: "StopMove",
    1004: "StandUp",
    1005: "StandDown",
    1006: "RecoveryStand",
    1007: "Euler",
    1008: "Move",
    1009: "Sit",
    1010: "RiseSit",
    1011: "SwitchGait",
    1012: "Trigger",
    1013: "BodyHeight",
    1014: "FootRaiseHeight",
    1015: "SpeedLevel",
    1016: "Hello",
    1017: "Stretch",
    1018: "TrajectoryFollow",
    1019: "ContinuousGait",
    1020: "Content",
    1021: "Wallow",
    1022: "Dance1",
    1023: "Dance2",
    1024: "GetBodyHeight",
    1025: "GetFootRaiseHeight",
    1026: "GetSpeedLevel",
    1027: "SwitchJoystick",
    1028: "Pose",
    1029: "Scrape",
    1030: "FrontFlip",
    1031: "FrontJump",
    1032: "FrontPounce",
    1033: "WiggleHips",
    1034: "GetState",
    1035: "EconomicGait",
    1036: "FingerHeart",
}
```

## Practical Examples

### Example 1: Keyboard Control

From `index.js`:

```196:236:javascript/index.js
  document.addEventListener('keydown', function(event) {
    const key = event.key.toLowerCase();
    let x = 0, y = 0, z = 0;

    switch (key) {
        case 'w': // Forward
            x = 0.8;
            break;
        case 's': // Reverse
            x = -0.4;
            break;
        case 'a': // Sideways left
            y = 0.4;
            break;
        case 'd': // Sideways right
            y = -0.4;
            break;
        case 'q': // Turn left
            z = 2;
            break;
        case 'e': // Turn right
            z = -2;
            break;
        default:
            return; // Ignore other keys
    }

    if(globalThis.rtc !== undefined) {
        globalThis.rtc.publishApi("rt/api/sport/request", 1008, JSON.stringify({x: x, y: y, z: z}));
    }
});
```

### Example 2: Gamepad Control

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

### Example 3: Python Joystick Example

```48:62:python/examples/joystick/go2_joystick.py
def gen_mov_command(x: float, y: float, z: float):
    x = x * JOY_SENSE
    y = y * JOY_SENSE

    command = {
        "type": "msg",
        "topic": "rt/api/sport/request",
        "data": {
            "header": {"identity": {"id": Go2Connection.generate_id(), "api_id": 1008}},
            "parameter": json.dumps({"x": x, "y": y, "z": z}),
        },
    }
    command = json.dumps(command)
    return command
```

## Message ID Generation

Unique IDs are generated using timestamp + random:

```324:327:python/go2_webrtc/go2_connection.py
    @staticmethod
    def generate_id():
        return int(
            datetime.datetime.now().timestamp() * 1000 % 2147483648
        ) + random.randint(0, 999)
```

Formula: `(timestamp_ms % 2^31) + random(0-999)`

## Error Handling

Currently, errors are silently caught in many places. The TODO comments indicate callbacks aren't fully implemented:

```148:149:javascript/go2webrtc.js
      // TODO: execute all the registred callbacks in a map defined
      // in the initRTC function
```

## Next Steps

- Read [Robot Control API](./05-robot-control-api.md) for detailed command information
- Check [Video Streaming](./06-video-streaming.md) to understand video handling
- See [LIDAR Data Decoding](./07-lidar-decoding.md) for sensor data

