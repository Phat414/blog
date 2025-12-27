---
title: "Thiết kế Custom Network Protocol"
date: 2025-11-28
description: "Cách thiết kế protocol riêng cho ứng dụng của bạn. Message format, serialization và protocol versioning..."
image: "https://images.unsplash.com/photo-1504639725590-34d0984388bd?w=800&h=500&fit=crop"
---

Khi HTTP hoặc các protocols chuẩn không đáp ứng được requirements, bạn có thể thiết kế custom protocol riêng. Bài viết này sẽ hướng dẫn cách thiết kế protocol hiệu quả.

## Why Custom Protocols?

### Use Cases

- **Gaming**: Low latency, high frequency updates
- **IoT**: Bandwidth constraints, custom requirements
- **Financial Systems**: High performance, custom message formats
- **Industrial Control**: Specialized communication needs

## Protocol Design Principles

### Key Considerations

1. **Efficiency**: Minimize overhead và bandwidth
2. **Reliability**: Error detection và recovery
3. **Extensibility**: Support future changes
4. **Security**: Authentication và encryption
5. **Simplicity**: Easy to implement và debug

## Message Format Design

### Binary vs Text

**Binary Protocol:**
- More efficient (smaller size)
- Faster parsing
- Harder to debug

**Text Protocol:**
- Human readable
- Easier to debug
- Larger size

### Message Structure

```
+--------+--------+----------+----------+
| Header | Length |   Type   | Payload  |
+--------+--------+----------+----------+
  2 bytes  4 bytes   2 bytes   Variable
```

## Example: Simple Binary Protocol

### Protocol Specification

```java
public class MessageProtocol {
    // Message types
    public static final short TYPE_HELLO = 1;
    public static final short TYPE_DATA = 2;
    public static final short TYPE_ACK = 3;
    public static final short TYPE_ERROR = 4;
    
    // Message structure
    public static class Message {
        private short magic = 0x4D53; // "MS" in hex
        private int length;
        private short type;
        private byte[] payload;
        
        public byte[] serialize() throws IOException {
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            DataOutputStream dos = new DataOutputStream(baos);
            
            dos.writeShort(magic);
            dos.writeInt(payload.length);
            dos.writeShort(type);
            dos.write(payload);
            
            return baos.toByteArray();
        }
        
        public static Message deserialize(byte[] data) throws IOException {
            ByteArrayInputStream bais = new ByteArrayInputStream(data);
            DataInputStream dis = new DataInputStream(bais);
            
            Message msg = new Message();
            msg.magic = dis.readShort();
            
            if (msg.magic != 0x4D53) {
                throw new IOException("Invalid magic number");
            }
            
            msg.length = dis.readInt();
            msg.type = dis.readShort();
            msg.payload = new byte[msg.length];
            dis.readFully(msg.payload);
            
            return msg;
        }
    }
}
```

## Text-Based Protocol

### Line-Based Protocol

```
HELLO username\r\n
DATA length:100 type:json\r\n
{payload data}\r\n
ACK message_id:123\r\n
ERROR code:500 message:Internal error\r\n
```

### Implementation

```javascript
class LineProtocol {
    constructor(socket) {
        this.socket = socket;
        this.buffer = '';
    }
    
    onData(data) {
        this.buffer += data.toString();
        
        let lines = this.buffer.split('\r\n');
        this.buffer = lines.pop(); // Keep incomplete line
        
        lines.forEach(line => this.processLine(line));
    }
    
    processLine(line) {
        const parts = line.split(' ');
        const command = parts[0];
        
        switch (command) {
            case 'HELLO':
                this.handleHello(parts[1]);
                break;
            case 'DATA':
                this.handleData(parts.slice(1));
                break;
            case 'ACK':
                this.handleAck(parts.slice(1));
                break;
            default:
                this.sendError(`Unknown command: ${command}`);
        }
    }
    
    send(command, ...args) {
        const message = [command, ...args].join(' ') + '\r\n';
        this.socket.write(message);
    }
}
```

## Protocol Versioning

### Version Negotiation

```java
public class ProtocolVersion {
    private int major;
    private int minor;
    
    public static final ProtocolVersion V1_0 = new ProtocolVersion(1, 0);
    public static final ProtocolVersion V2_0 = new ProtocolVersion(2, 0);
    
    public boolean isCompatible(ProtocolVersion other) {
        return this.major == other.major;
    }
    
    // Handshake
    public static ProtocolVersion negotiate(ProtocolVersion client, 
                                           ProtocolVersion server) {
        if (client.major == server.major) {
            return new ProtocolVersion(
                client.major,
                Math.min(client.minor, server.minor)
            );
        }
        throw new IncompatibleVersionException();
    }
}
```

## Serialization Formats

### JSON

```javascript
const message = {
    type: 'DATA',
    timestamp: Date.now(),
    payload: {
        user: 'john',
        action: 'login'
    }
};

const json = JSON.stringify(message);
socket.write(json + '\n');
```

### Protocol Buffers

```protobuf
syntax = "proto3";

message NetworkMessage {
    enum MessageType {
        HELLO = 0;
        DATA = 1;
        ACK = 2;
    }
    
    MessageType type = 1;
    int64 timestamp = 2;
    bytes payload = 3;
}
```

### MessagePack

```javascript
const msgpack = require('msgpack5')();

const message = {
    type: 'DATA',
    data: { user: 'john', score: 100 }
};

const encoded = msgpack.encode(message);
socket.write(encoded);
```

## Error Handling

### Error Detection

```java
public class CRC32Checksum {
    public static byte[] addChecksum(byte[] data) {
        CRC32 crc = new CRC32();
        crc.update(data);
        long checksum = crc.getValue();
        
        ByteBuffer buffer = ByteBuffer.allocate(data.length + 4);
        buffer.put(data);
        buffer.putInt((int) checksum);
        
        return buffer.array();
    }
    
    public static boolean verify(byte[] dataWithChecksum) {
        ByteBuffer buffer = ByteBuffer.wrap(dataWithChecksum);
        
        byte[] data = new byte[dataWithChecksum.length - 4];
        buffer.get(data);
        int receivedChecksum = buffer.getInt();
        
        CRC32 crc = new CRC32();
        crc.update(data);
        
        return receivedChecksum == (int) crc.getValue();
    }
}
```

## Flow Control

### Sliding Window Protocol

```java
public class SlidingWindow {
    private final int windowSize;
    private int sendBase = 0;
    private int nextSeqNum = 0;
    private Map<Integer, byte[]> unacknowledged = new HashMap<>();
    
    public boolean canSend() {
        return nextSeqNum < sendBase + windowSize;
    }
    
    public void send(byte[] data) {
        if (!canSend()) {
            throw new IllegalStateException("Window is full");
        }
        
        unacknowledged.put(nextSeqNum, data);
        sendPacket(nextSeqNum, data);
        nextSeqNum++;
    }
    
    public void acknowledgeUpTo(int seqNum) {
        while (sendBase <= seqNum) {
            unacknowledged.remove(sendBase);
            sendBase++;
        }
    }
    
    public void retransmit() {
        for (int i = sendBase; i < nextSeqNum; i++) {
            sendPacket(i, unacknowledged.get(i));
        }
    }
}
```

## Connection Management

### Heartbeat Mechanism

```javascript
class ConnectionManager {
    constructor(socket) {
        this.socket = socket;
        this.lastHeartbeat = Date.now();
        this.heartbeatInterval = 5000; // 5 seconds
        this.timeout = 15000; // 15 seconds
        
        this.startHeartbeat();
        this.startTimeoutCheck();
    }
    
    startHeartbeat() {
        this.heartbeatTimer = setInterval(() => {
            this.sendHeartbeat();
        }, this.heartbeatInterval);
    }
    
    startTimeoutCheck() {
        this.timeoutTimer = setInterval(() => {
            const now = Date.now();
            if (now - this.lastHeartbeat > this.timeout) {
                this.onTimeout();
            }
        }, 1000);
    }
    
    sendHeartbeat() {
        this.socket.write('PING\r\n');
    }
    
    receiveHeartbeat() {
        this.lastHeartbeat = Date.now();
        this.socket.write('PONG\r\n');
    }
    
    onTimeout() {
        console.log('Connection timeout');
        this.socket.destroy();
        clearInterval(this.heartbeatTimer);
        clearInterval(this.timeoutTimer);
    }
}
```

## Security Considerations

### Authentication

```java
public class ProtocolAuthentication {
    public static class AuthChallenge {
        private byte[] nonce;
        
        public AuthChallenge() {
            this.nonce = generateNonce();
        }
        
        private byte[] generateNonce() {
            byte[] nonce = new byte[16];
            new SecureRandom().nextBytes(nonce);
            return nonce;
        }
        
        public boolean verify(String username, byte[] response, String password) {
            byte[] expected = computeResponse(username, password, nonce);
            return MessageDigest.isEqual(expected, response);
        }
        
        private byte[] computeResponse(String username, String password, byte[] nonce) {
            try {
                MessageDigest md = MessageDigest.getInstance("SHA-256");
                md.update(username.getBytes());
                md.update(password.getBytes());
                md.update(nonce);
                return md.digest();
            } catch (NoSuchAlgorithmException e) {
                throw new RuntimeException(e);
            }
        }
    }
}
```

## Testing

### Protocol Fuzzing

```python
import socket
import random

def fuzz_test(host, port, iterations=1000):
    for i in range(iterations):
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.connect((host, port))
        
        # Send random data
        data = bytes([random.randint(0, 255) for _ in range(random.randint(1, 1024))])
        sock.send(data)
        
        # Check if server crashes
        try:
            response = sock.recv(1024)
        except:
            print(f"Server error on iteration {i}")
        
        sock.close()
```

## Best Practices

1. **Document Everything**: Detailed protocol specification
2. **Version from Day One**: Plan for protocol evolution
3. **Use Standard Serialization**: JSON, Protobuf, MessagePack
4. **Implement Timeouts**: Prevent hanging connections
5. **Add Checksums**: Detect data corruption
6. **Test Thoroughly**: Unit tests, integration tests, fuzzing
7. **Monitor Performance**: Latency, throughput, error rates
8. **Keep it Simple**: Start simple, add complexity as needed

## Kết luận

Thiết kế custom protocol là một skill quan trọng khi building network applications. Hiểu rõ các principles và best practices sẽ giúp bạn tạo ra protocols hiệu quả, reliable và maintainable.
