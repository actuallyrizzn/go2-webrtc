# Security & Authentication

This document explains the security mechanisms used to securely connect and authenticate with the Unitree GO2 robot.

## Security Architecture

The system uses multiple layers of security:

1. **Authentication Tokens** - MD5-hashed tokens for ongoing validation
2. **RSA Public Key Encryption** - For secure key exchange
3. **AES Symmetric Encryption** - For encrypting large data payloads
4. **Hybrid Encryption** - Combines RSA and AES for efficiency

## Authentication Tokens

### What is a Token?

A token is a credential that identifies you to the robot. It looks like:
```
eyJ0eXAiOizI1NiJtlbiI[..]CI6MTcwODAxMzUwOX0.hiWOd9tNCIPzOOLNA
```

### Obtaining a Token

#### Method 1: Extract from Phone App Traffic

The token can be captured by sniffing network traffic between your phone and the robot:

```bash
# Using ngrep (Linux/Mac)
ngrep port 8081

# Using Wireshark
# Filter: tcp.port == 8081
```

Look for the token in the initial connection payload:
```json
{
  "token": "eyJ0eXAi...",
  "sdp": "v=0\r\no=- ",
  "id": "STA_localNetwork",
  "type": "offer"
}
```

#### Method 2: API Endpoint

```bash
curl -vX POST https://global-robot-api.unitree.com/login/email \
  -d "email=<EMAIL>&password=<MD5_HASH_OF_PASSWORD>"
```

Note: The password must be MD5 hashed before sending!

#### Method 3: Tokenless Connection

You can connect **without** a token, but with limitations:
- Only **one active connection** at a time
- Multiple clients require a valid token

### Token Validation Flow

After the WebRTC connection is established, the robot sends a validation challenge:

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
1. Robot sends a challenge string (e.g., `"abc123"`)
2. Client encrypts it using `encryptKey()`
3. Sends back the encrypted version
4. Robot validates and responds `"Validation Ok."`

### Token Encryption Function

```306:322:python/go2_webrtc/go2_connection.py
    @staticmethod
    def encrypt_key(key):
        # Append the prefix to the key
        prefixed_key = f"UnitreeGo2_{key}"
        # Encrypt the key using MD5 and convert to hex string
        encrypted = Go2Connection.encrypt_by_md5(prefixed_key)
        # Convert the hex string to Base64
        return Go2Connection.hex_to_base64(encrypted)

    @staticmethod
    def encrypt_by_md5(input_str):
        # Create an MD5 hash object
        hash_obj = hashlib.md5()
        # Update the hash object with the bytes of the input string
        hash_obj.update(input_str.encode("utf-8"))
        # Return the hex digest of the hash
        return hash_obj.hexdigest()
```

**JavaScript version:**

```11:19:javascript/utils.js
export const encryptKey = (r) => {
  const n = `UnitreeGo2_${r}`,
    o = encryptByMd5(n);
  return hexToBase64(o);
};

function encryptByMd5(r) {
  return md5(r).toString();
}
```

**Algorithm:**
1. Prepend `"UnitreeGo2_"` to the challenge
2. Compute MD5 hash
3. Convert hex to Base64
4. Return Base64-encoded result

**Example:**
```python
challenge = "abc123"
prefixed = "UnitreeGo2_abc123"
md5_hash = md5(prefixed)  # e.g., "3d5f8a9b2c1e..."
hex = md5_hash.hexdigest()
base64 = base64.b64encode(hex)  # Final encrypted value
```

## RSA Encryption (Key Exchange)

### Why RSA?

RSA is an asymmetric encryption algorithm. It uses a **public key** (known to everyone) and a **private key** (secret). The robot's public key can be used to encrypt data that only the robot can decrypt.

### Getting the Robot's Public Key

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
```

The robot responds with:
```json
{
  "data1": "-----BEGIN PUBLIC KEY-----\n<base64-encoded key>\n-----END PUBLIC KEY-----"
}
```

### Loading and Using the Public Key

```386:389:python/go2_webrtc/go2_connection.py
    @staticmethod
    def rsa_load_public_key(pem_data: str) -> RSA.RsaKey:
        """Load an RSA public key from a PEM-formatted string."""
        key_bytes = base64.b64decode(pem_data)
        return RSA.import_key(key_bytes)
```

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

**Why chunking?** RSA can only encrypt blocks of a certain size (key size - 11 bytes for PKCS1 padding). Longer data must be split.

## AES Encryption (Data Encryption)

### Why AES?

AES is much faster than RSA for encrypting large amounts of data. We use AES to encrypt the entire SDP offer, then use RSA to securely transmit the AES key.

### Generating an AES Key

```380:383:python/go2_webrtc/go2_connection.py
    @staticmethod
    def generate_aes_key() -> str:
        uuid_32 = uuid.uuid4().bytes
        uuid_32_hex_string = binascii.hexlify(uuid_32).decode("utf-8")
        return uuid_32_hex_string
```

Creates a 32-byte (256-bit) random key in hex format.

### AES Encryption Process

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

**Padding:**
```392:397:python/go2_webrtc/go2_connection.py
    @staticmethod
    def pad(data: str) -> bytes:
        """Pad data to be a multiple of 16 bytes (AES block size)."""
        block_size = AES.block_size
        padding = block_size - len(data) % block_size
        padded_data = data + chr(padding) * padding
        return padded_data.encode("utf-8")
```

AES requires data to be a multiple of 16 bytes. If not, we append padding bytes (PKCS5).

### AES Decryption

```435:447:python/go2_webrtc/go2_connection.py
    @staticmethod
    def aes_decrypt(encrypted_data: str, key: str) -> str:
        """Decrypt the given data using AES (ECB mode with PKCS5 padding)."""
        # Ensure key is 32 bytes for AES-256
        key_bytes = key.encode("utf-8")
        # Decode Base64 encrypted data
        encrypted_data_bytes = base64.b64decode(encrypted_data)
        # Create AES cipher in ECB mode
        cipher = AES.new(key_bytes, AES.MODE_ECB)
        # Decrypt data
        decrypted_padded_data = cipher.decrypt(encrypted_data_bytes)
        # Unpad the decrypted data
        decrypted_data = Go2Connection.unpad(decrypted_padded_data)
        return decrypted_data
```

```429:432:python/go2_webrtc/go2_connection.py
    @staticmethod
    def unpad(data: bytes) -> str:
        """Remove padding from data."""
        padding = data[-1]
        return data[:-padding].decode("utf-8")
```

## Hybrid Encryption in Action

Here's how the hybrid encryption works for the SDP handshake:

```python
# Step 1: Generate a random AES key
aes_key = generate_aes_key()  # e.g., "a1b2c3d4e5f6..."

# Step 2: Encrypt the SDP offer with AES
encrypted_sdp = aes_encrypt(json.dumps(sdp_offer), aes_key)

# Step 3: Get robot's public key
robot_pub_key = get_robot_public_key()

# Step 4: Encrypt the AES key with RSA
encrypted_aes_key = rsa_encrypt(aes_key, robot_pub_key)

# Step 5: Send both encrypted values to robot
payload = {
    "data1": encrypted_sdp,        # AES-encrypted SDP
    "data2": encrypted_aes_key    # RSA-encrypted AES key
}
```

**Why this approach?**
- RSA is slow for large data → Use for small AES key only
- AES is fast for large data → Use for SDP offer
- Best of both worlds!

## Base64 Encoding

Many of the encrypted values are Base64-encoded for safe transmission over HTTP:

```299:303:python/go2_webrtc/go2_connection.py
    @staticmethod
    def hex_to_base64(hex_str):
        # Convert hex string to bytes
        bytes_array = bytes.fromhex(hex_str)
        # Encode the bytes to Base64 and return as a string
        return base64.b64encode(bytes_array).decode("utf-8")
```

## Security Considerations

### What This System Does Well

1. ✅ Uses established cryptographic algorithms (AES, RSA)
2. ✅ Implements proper key exchange
3. ✅ Encrypts sensitive data (SDP)
4. ✅ Uses token-based authentication

### Potential Vulnerabilities

1. ⚠️ **MD5 is considered weak** - But used only for token validation, not password storage
2. ⚠️ **No certificate validation** - Robot's public key is trusted without verification
3. ⚠️ **ECB mode** - AES-ECB is not ideal (no IV), but used consistently by robot's API
4. ⚠️ **Token extraction** - Tokens can be sniffed from network traffic

### Best Practices for Users

1. **Keep tokens secret** - Don't commit them to version control
2. **Use HTTPS when possible** - Though this code uses HTTP (local network)
3. **Rotate tokens periodically** - If you regenerate them
4. **Monitor connections** - Watch for unauthorized access

## Next Steps

- Read [Data Channels & Messaging](./04-data-channels-messaging.md) to understand the communication protocol
- Check [Robot Control API](./05-robot-control-api.md) to learn about commands

