# LIDAR Data Decoding

This document explains how compressed LIDAR voxel data is decoded using WebAssembly.

## What is LIDAR Data?

The Unitree GO2 robot includes a LIDAR sensor that generates 3D point clouds representing the environment. This data is:
- **Compressed** for efficient transmission
- **Encoded as voxels** (3D pixels)
- **Decoded client-side** using WebAssembly

## LIDAR Data Structure

### Binary Format

LIDAR data arrives as a binary ArrayBuffer with this structure:

```
[2 bytes: length][2 bytes: ?][length bytes: JSON metadata][remaining: compressed data]
```

**Fields:**
- First 2 bytes: Length of JSON metadata
- Next 2 bytes: Unknown/unused?
- JSON metadata: Contains information about the compressed data
- Remaining bytes: Compressed voxel data

### JSON Metadata Example

```json
{
  "data": {
    "origin": [x, y, z],
    "resolution": 0.05,
    "data": <reference to binary data>
  }
}
```

## WASM-Based Decoding

The decoder uses **WebAssembly** (WASM) for fast, native-speed decompression.

### Why WebAssembly?

- **Performance**: Near-native speed for computational tasks
- **Cross-platform**: Works in browsers and Python
- **Access to native code**: Can use optimized C/C++ decompression algorithms

### WASM Module

The decoder loads `libvoxel.wasm`:

```32:42:python/go2_webrtc/lidar_decoder.py
class LidarDecoder:
    def __init__(self) -> None:

        config = Config()
        config.wasm_multi_value = True
        config.debug_info = True
        self.store = Store(Engine(config))

        module_dir = os.path.dirname(os.path.abspath(__file__))

        self.module = Module.from_file(self.store.engine, os.path.join(module_dir,"libvoxel.wasm"))
```

This WASM module contains optimized decompression functions.

## Decoding Process

### 1. Parse Binary Header

```330:336:python/go2_webrtc/go2_connection.py
    @staticmethod
    def deal_array_buffer(n):
        # Unpack the first 2 bytes as an unsigned short (16-bit) to get the length
        length = struct.unpack("H", n[:2])[0]

        # Extract the JSON segment and the remaining data
        json_segment = n[4 : 4 + length]
```

**JavaScript version:**

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

### 2. Extract Metadata and Compressed Data

```337:340:python/go2_webrtc/go2_connection.py
        remaining_data = n[4 + length :]

        # Decode the JSON segment from UTF-8 and parse it
        json_str = json_segment.decode("utf-8")
        obj = json.loads(json_str)
```

### 3. Decompress with WASM

```342:347:python/go2_webrtc/go2_connection.py
        decoded_data = decoder.decode(remaining_data, obj["data"])

        # Attach the remaining data to the object
        obj["data"]["data"] = decoded_data

        return obj
```

The WASM decoder performs the actual decompression:

```120:162:python/go2_webrtc/lidar_decoder.py
    def decode(self, compressed_data, data):
        self.add_value_arr(self.input, compressed_data)

        some_v = math.floor(data["origin"][2] / data["resolution"])

        self.generate(
            self.store,
            self.input,
            len(compressed_data),
            self.decompressBufferSize,
            self.decompressBuffer,
            self.decompressedSize, 
            self.positions,
            self.uvs,
            self.indices,          
            self.faceCount,
            self.pointCount,        
            some_v
        )

        self.get_value(self.decompressedSize, "i32")
        c = self.get_value(self.pointCount, "i32")
        u = self.get_value(self.faceCount, "i32")

        positions_slice = self.HEAPU8[self.positions:self.positions + u * 12]
        positions_copy = bytearray(positions_slice)
        p = np.frombuffer(positions_copy, dtype=np.uint8)

        uvs_slice = self.HEAPU8[self.uvs:self.uvs + u * 8]
        uvs_copy = bytearray(uvs_slice)
        r = np.frombuffer(uvs_copy, dtype=np.uint8)

        indices_slice = self.HEAPU8[self.indices:self.indices + u * 24]
        indices_copy = bytearray(indices_slice)
        o = np.frombuffer(indices_copy, dtype=np.uint32)

        return {
            "point_count": c,
            "face_count": u,
            "positions": p,
            "uvs": r,
            "indices": o
        }
```

## WASM Memory Management

The decoder interacts with WASM memory through typed arrays:

```56:68:python/go2_webrtc/lidar_decoder.py
        self.buffer_ptr = int.from_bytes(self.buffer, "little")

        self.HEAP8 = (ctypes.c_int8 * self.memory_size).from_address(self.buffer_ptr)
        self.HEAP16 = (ctypes.c_int16 * (self.memory_size // 2)).from_address(self.buffer_ptr)
        self.HEAP32 = (ctypes.c_int32 * (self.memory_size // 4)).from_address(self.buffer_ptr)
        self.HEAPU8 = (ctypes.c_uint8 * self.memory_size).from_address(self.buffer_ptr)
        self.HEAPU16 = (ctypes.c_uint16 * (self.memory_size // 2)).from_address(self.buffer_ptr)
        self.HEAPU32 = (ctypes.c_uint32 * (self.memory_size // 4)).from_address(self.buffer_ptr)
        self.HEAPF32 = (ctypes.c_float * (self.memory_size // 4)).from_address(self.buffer_ptr)
        self.HEAPF64 = (ctypes.c_double * (self.memory_size // 8)).from_address(self.buffer_ptr)
```

These typed arrays allow direct access to WASM memory as Python arrays.

### WASM Callbacks

The decoder provides callbacks to the WASM module:

```46:49:python/go2_webrtc/lidar_decoder.py
        a = Func(self.store, self.a_callback_type, self.adjust_memory_size)
        b = Func(self.store, self.b_callback_type, self.copy_memory_region)

        self.instance = Instance(self.store, self.module, [a, b])
```

```80:93:python/go2_webrtc/lidar_decoder.py
    def adjust_memory_size(self, t):
        return len(self.HEAPU8)

    def copy_within(self, target, start, end):
        # Copy the sublist for the specified range [start:end]
        sublist = self.HEAPU8[start:end]

        # Replace elements in the list starting from index 'target'
        for i in range(len(sublist)):
            if target + i < len(self.HEAPU8):
                self.HEAPU8[target + i] = sublist[i]
    
    def copy_memory_region(self, t, n, a):
        self.copy_within(t, n, n + a)
```

## Decoded Data Format

The decoder returns a dictionary with:

```python
{
    "point_count": <number>,      # Number of 3D points
    "face_count": <number>,       # Number of triangular faces
    "positions": <numpy_array>,   # Vertex positions (x, y, z)
    "uvs": <numpy_array>,         # UV texture coordinates
    "indices": <numpy_array>      # Face indices (triangles)
}
```

### Data Types

- **positions**: `uint8` array (3 bytes per vertex)
- **uvs**: `uint8` array (2 bytes per UV coordinate)
- **indices**: `uint32` array (3 indices per triangle)

## Memory Allocation

The decoder pre-allocates buffers for decompression:

```70:78:python/go2_webrtc/lidar_decoder.py
        self.input = self.malloc(self.store, 61440)
        self.decompressBuffer = self.malloc(self.store, 80000)
        self.positions = self.malloc(self.store, 2880000)
        self.uvs = self.malloc(self.store, 1920000)
        self.indices = self.malloc(self.store, 5760000)
        self.decompressedSize = self.malloc(self.store, 4)
        self.faceCount = self.malloc(self.store, 4)
        self.pointCount = self.malloc(self.store, 4)
```

These buffers are reused for each decompression operation.

## Using Decoded Data

### Python Example

```python
from go2_webrtc import Go2Connection

def handle_lidar(message):
    if isinstance(message, bytes):
        obj = Go2Connection.deal_array_buffer(message)
        decoded = obj["data"]["data"]  # Already decoded
        
        print(f"Points: {decoded['point_count']}")
        print(f"Faces: {decoded['face_count']}")
        # positions, uvs, indices are numpy arrays

conn = Go2Connection(ip, token, on_message=handle_lidar)
```

### Visualization

The decoded data can be visualized using Three.js (see `javascript/threejs.html`):
- Points: Plot 3D points
- Mesh: Render triangular faces
- Texture: Apply UV mapping

## Performance Considerations

### Decoding Speed

- WASM decompression is **very fast** (near-native speed)
- Typical decompression: **<10ms** for standard density point clouds
- Main bottleneck: **Network transfer time**

### Memory Usage

- Decoded point clouds use **significant memory**
- Typical: **1-10 MB** per frame
- Consider streaming/buffering strategy for large datasets

### Compression Ratio

- Typical compression: **10:1 to 100:1**
- Depends on point cloud density and distribution
- Sparse clouds compress better than dense ones

## Accessing LIDAR Data

### Subscribe to LIDAR Topic

```javascript
// Subscribe to compressed LIDAR stream
globalThis.rtc.publish("rt/utlidar/voxel_map_compressed", "", 
                       DataChannelType.SUBSCRIBE);
```

### Receive LIDAR Messages

Messages will arrive as binary ArrayBuffers when subscribed. The data channel handler will process them:

```javascript
handleDataChannelMessage(event) {
    const data = typeof event.data == "string"
        ? JSON.parse(event.data)
        : this.dealArrayBuffer(event.data);  // LIDAR uses binary path
    // ...
}
```

## Troubleshooting

### Decoding Fails

- **Check**: Is `libvoxel.wasm` present?
- **Error**: "Module not found" → File path issue
- **Error**: "Memory allocation failed" → Buffer too small

### Missing LIDAR Data

- **Check**: Are you subscribed to the right topic?
- **Check**: Does robot have LIDAR enabled?
- **Check**: Is LIDAR hardware working?

## Advanced: Custom Decoding

To implement custom decoding:

1. Load WASM module
2. Provide memory buffers
3. Call WASM functions
4. Extract results from memory

See `lidar_decoder.py` for reference implementation.

## Next Steps

- Read [Code Walkthrough](./08-code-walkthrough.md) for implementation details
- Check [System Architecture](./01-architecture-overview.md) for the big picture
- Review [Data Channels](./04-data-channels-messaging.md) for messaging protocol

