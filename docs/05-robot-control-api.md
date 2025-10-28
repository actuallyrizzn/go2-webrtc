# Robot Control API

This document explains how to control the Unitree GO2 robot using the available APIs and commands.

## API Overview

The robot control system uses a topic-based messaging system with numeric API IDs. All commands are sent via the data channel on the topic `rt/api/sport/request`.

## Command Structure

Every command follows this structure:

```json
{
  "type": "msg",
  "topic": "rt/api/sport/request",
  "data": {
    "header": {
      "identity": {
        "id": <unique_message_id>,
        "api_id": <command_id>
      }
    },
    "parameter": "<json_stringified_data>"
  }
}
```

## Sport Commands Reference

### Movement Commands

#### Move (1008)
Control robot movement in 3D space.

**Parameters:**
- `x`: Forward/backward velocity (-1.0 to 1.0)
- `y`: Sideways velocity (-1.0 to 1.0)
- `z`: Rotational velocity (negative = left, positive = right)

**Example:**
```javascript
// Move forward
globalThis.rtc.publishApi("rt/api/sport/request", 1008, 
  JSON.stringify({x: 0.8, y: 0, z: 0}));

// Move diagonally forward-right while turning
globalThis.rtc.publishApi("rt/api/sport/request", 1008, 
  JSON.stringify({x: 0.6, y: -0.4, z: -1.5}));

// Stop
globalThis.rtc.publishApi("rt/api/sport/request", 1008, 
  JSON.stringify({x: 0, y: 0, z: 0}));
```

#### Euler (1007)
Control robot orientation.

**Parameters:**
- `roll`, `pitch`, `yaw`: Orientation angles in radians

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1007, 
  JSON.stringify({roll: 0, pitch: 0.1, yaw: 0}));
```

### State Commands

#### BalanceStand (1002)
Stand upright in balance mode.

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1002, 
  JSON.stringify(1002));
```

#### Sit (1009)
Sit down.

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1009, 
  JSON.stringify(1009));
```

#### StandUp (1004)
Stand up from sitting or lying position.

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1004, 
  JSON.stringify(1004));
```

#### StandDown (1005)
Lie down.

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1005, 
  JSON.stringify(1005));
```

#### StopMove (1003)
Stop all movement immediately.

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1003, 
  JSON.stringify(1003));
```

#### Damp (1001)
Dampen the robot (soften/hold position).

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1001, 
  JSON.stringify(1001));
```

#### RecoveryStand (1006)
Recovery standing mode (for when robot falls).

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1006, 
  JSON.stringify(1006));
```

### Body Adjustment Commands

#### BodyHeight (1013)
Adjust body height.

**Parameters:**
- Value: Body height setting

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1013, 
  JSON.stringify(0.2)); // 0.2 meter body height
```

#### FootRaiseHeight (1014)
Adjust foot raise height.

**Parameters:**
- Value: Foot raise height

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1014, 
  JSON.stringify(0.05)); // 5cm foot raise
```

#### SpeedLevel (1015)
Set movement speed level.

**Parameters:**
- Value: Speed level

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1015, 
  JSON.stringify(3)); // Level 3 speed
```

### Gait Commands

#### SwitchGait (1011)
Switch between different gaits.

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1011, 
  JSON.stringify(1011));
```

#### ContinuousGait (1019)
Enable continuous gait mode.

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1019, 
  JSON.stringify(1019));
```

#### EconomicGait (1035)
Enable economic (energy-efficient) gait.

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1035, 
  JSON.stringify(1035));
```

### Trick & Behavior Commands

#### Hello (1016)
Perform a greeting gesture.

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1016, 
  JSON.stringify(1016));
```

#### Stretch (1017)
Stretch routine.

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1017, 
  JSON.stringify(1017));
```

#### Dance1 (1022)
Perform dance routine 1.

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1022, 
  JSON.stringify(1022));
```

#### Dance2 (1023)
Perform dance routine 2.

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1023, 
  JSON.stringify(1023));
```

#### FrontFlip (1030)
Perform a front flip.

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1030, 
  JSON.stringify(1030));
```

#### FrontJump (1031)
Perform a front jump.

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1031, 
  JSON.stringify(1031));
```

#### WiggleHips (1033)
Wiggle the hips.

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1033, 
  JSON.stringify(1033));
```

### Information Queries

#### GetState (1034)
Get current robot state.

**Response:** Returns robot status information.

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1034, 
  JSON.stringify(1034));
```

#### GetBodyHeight (1024)
Get current body height.

**Response:** Returns current height setting.

**Example:**
```javascript
globalThis.rtc.publishApi("rt/api/sport/request", 1024, 
  JSON.stringify(1024));
```

## Complete Command Reference

| ID | Command | Description | Parameters |
|----|---------|-------------|------------|
| 1001 | Damp | Dampen/hold position | None |
| 1002 | BalanceStand | Stand in balance mode | None |
| 1003 | StopMove | Stop all movement | None |
| 1004 | StandUp | Stand up | None |
| 1005 | StandDown | Lie down | None |
| 1006 | RecoveryStand | Recovery standing | None |
| 1007 | Euler | Control orientation | roll, pitch, yaw |
| 1008 | Move | Move robot | x, y, z velocities |
| 1009 | Sit | Sit down | None |
| 1010 | RiseSit | Stand up from sit | None |
| 1011 | SwitchGait | Change gait mode | None |
| 1012 | Trigger | Trigger action | Various |
| 1013 | BodyHeight | Set body height | height value |
| 1014 | FootRaiseHeight | Set foot height | height value |
| 1015 | SpeedLevel | Set speed level | level value |
| 1016 | Hello | Greeting gesture | None |
| 1017 | Stretch | Stretch routine | None |
| 1018 | TrajectoryFollow | Follow trajectory | trajectory data |
| 1019 | ContinuousGait | Enable continuous gait | None |
| 1020 | Content | Content mode | Various |
| 1021 | Wallow | Wallow behavior | None |
| 1022 | Dance1 | Dance routine 1 | None |
| 1023 | Dance2 | Dance routine 2 | None |
| 1024 | GetBodyHeight | Query body height | None |
| 1025 | GetFootRaiseHeight | Query foot height | None |
| 1026 | GetSpeedLevel | Query speed level | None |
| 1027 | SwitchJoystick | Enable joystick mode | None |
| 1028 | Pose | Set pose | pose data |
| 1029 | Scrape | Scrape behavior | None |
| 1030 | FrontFlip | Perform front flip | None |
| 1031 | FrontJump | Perform front jump | None |
| 1032 | FrontPounce | Perform front pounce | None |
| 1033 | WiggleHips | Wiggle hips | None |
| 1034 | GetState | Get robot state | None |
| 1035 | EconomicGait | Economic gait mode | None |
| 1036 | FingerHeart | Finger heart gesture | None |

## Usage Patterns

### Basic Control Loop

```javascript
// 1. Connect
globalThis.rtc = new Go2WebRTC(token, robotIP);
globalThis.rtc.initSDP();

// 2. Wait for validation
// (handled automatically)

// 3. Stand up
setTimeout(() => {
  globalThis.rtc.publishApi("rt/api/sport/request", 1004, 
    JSON.stringify(1004));
}, 2000);

// 4. Move
setTimeout(() => {
  globalThis.rtc.publishApi("rt/api/sport/request", 1008, 
    JSON.stringify({x: 0.5, y: 0, z: 0}));
}, 4000);

// 5. Stop
setTimeout(() => {
  globalThis.rtc.publishApi("rt/api/sport/request", 1008, 
    JSON.stringify({x: 0, y: 0, z: 0}));
}, 8000);
```

### Continuous Control (Game Loop)

```javascript
// From index.js - runs every 100ms
function joystickTick(joyLeft, joyRight) {
  let x, y, z = 0;
  
  // Read input devices
  y = -1 * (joyRight.GetPosX() - 100) / 50;
  x = -1 * (joyLeft.GetPosY() - 100) / 50;
  z = -1 * (joyLeft.GetPosX() - 100) / 50;
  
  // Only send if there's input
  if (x === 0 && y === 0 && z === 0) return;
  
  // Send command
  if(globalThis.rtc !== undefined) {
    globalThis.rtc.publishApi("rt/api/sport/request", 1008, 
      JSON.stringify({x, y, z}));
  }
}

setInterval(() => joystickTick(joyLeft, joyRight), 100);
```

### Python Example

```python
from go2_webrtc import Go2Connection
import asyncio
import json

async def control_robot():
    conn = Go2Connection(robot_ip, token)
    await conn.connect_robot()
    
    # Stand up
    conn.publish("rt/api/sport/request", 
                  json.dumps({"header": 
                              {"identity": 
                               {"id": Go2Connection.generate_id(), 
                                "api_id": 1004}}, 
                              "parameter": "1004"}))
    
    await asyncio.sleep(2)
    
    # Move forward
    conn.publish("rt/api/sport/request",
                 json.dumps({"header":
                            {"identity":
                             {"id": Go2Connection.generate_id(),
                              "api_id": 1008}},
                            "parameter": json.dumps({"x": 0.8, "y": 0, "z": 0})}))
    
    await asyncio.sleep(3)
    
    # Stop
    conn.publish("rt/api/sport/request",
                 json.dumps({"header":
                            {"identity":
                             {"id": Go2Connection.generate_id(),
                              "api_id": 1008}},
                            "parameter": json.dumps({"x": 0, "y": 0, "z": 0})}))
```

## Safety Considerations

1. **Start Slow**: Test commands with small values first
2. **Emergency Stop**: Keep 1003 (StopMove) ready
3. **Space Requirements**: Tricks like flips need adequate space
4. **Battery**: High-speed movements drain battery faster
5. **Overrides**: The robot may refuse dangerous commands

## Command Response Handling

Currently, command responses aren't fully implemented in the callback system. The TODO comments indicate planned improvements:

```javascript
// TODO: execute all the registred callbacks in a map defined
// in the initRTC function
```

To track responses, you could extend the `Go2WebRTC` class to store and match response IDs.

## Next Steps

- Review [Video Streaming](./06-video-streaming.md) to see how video is handled
- Check [LIDAR Data Decoding](./07-lidar-decoding.md) for sensor data
- Read [Code Walkthrough](./08-code-walkthrough.md) for implementation details

