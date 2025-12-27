---
title: "Xây dựng TCP Server với Node.js"
date: 2025-12-01
description: "Sử dụng Net module trong Node.js để tạo TCP server. Handle multiple clients, streaming data và error handling..."
image: "https://images.unsplash.com/photo-1550751827-4bd374c3f58b?w=800&h=500&fit=crop"
---

Node.js cung cấp Net module mạnh mẽ để xây dựng TCP servers và clients. Trong bài viết này, chúng ta sẽ tìm hiểu cách tạo TCP server với Node.js.

## Net Module Overview

Net module cung cấp asynchronous network API để tạo stream-based TCP servers và clients.

### Basic TCP Server

```javascript
const net = require('net');

const server = net.createServer((socket) => {
    console.log('Client connected');
    
    socket.on('data', (data) => {
        console.log('Received:', data.toString());
        socket.write(`Echo: ${data}`);
    });
    
    socket.on('end', () => {
        console.log('Client disconnected');
    });
    
    socket.on('error', (err) => {
        console.error('Socket error:', err);
    });
});

server.listen(8080, () => {
    console.log('Server listening on port 8080');
});
```

## TCP Client

### Connecting to Server

```javascript
const net = require('net');

const client = net.createConnection({ port: 8080 }, () => {
    console.log('Connected to server');
    client.write('Hello Server!');
});

client.on('data', (data) => {
    console.log('Server response:', data.toString());
    client.end();
});

client.on('end', () => {
    console.log('Disconnected from server');
});

client.on('error', (err) => {
    console.error('Connection error:', err);
});
```

## Handling Multiple Clients

### Connection Pool

```javascript
const net = require('net');

const connections = new Set();

const server = net.createServer((socket) => {
    connections.add(socket);
    console.log(`Client connected. Total: ${connections.size}`);
    
    socket.on('data', (data) => {
        // Broadcast to all clients
        const message = data.toString();
        connections.forEach((client) => {
            if (client !== socket) {
                client.write(`[Broadcast] ${message}`);
            }
        });
    });
    
    socket.on('end', () => {
        connections.delete(socket);
        console.log(`Client disconnected. Total: ${connections.size}`);
    });
    
    socket.on('error', (err) => {
        console.error('Socket error:', err);
        connections.delete(socket);
    });
});

server.listen(8080);
```

## Streaming Data

### File Transfer Server

```javascript
const net = require('net');
const fs = require('fs');

const server = net.createServer((socket) => {
    socket.on('data', (data) => {
        const filename = data.toString().trim();
        
        if (!fs.existsSync(filename)) {
            socket.write('ERROR: File not found\n');
            socket.end();
            return;
        }
        
        const readStream = fs.createReadStream(filename);
        
        readStream.on('data', (chunk) => {
            if (!socket.write(chunk)) {
                readStream.pause();
            }
        });
        
        socket.on('drain', () => {
            readStream.resume();
        });
        
        readStream.on('end', () => {
            socket.end();
        });
        
        readStream.on('error', (err) => {
            socket.write(`ERROR: ${err.message}\n`);
            socket.end();
        });
    });
});

server.listen(8080);
```

## Protocol Implementation

### Simple Chat Protocol

```javascript
const net = require('net');

class ChatServer {
    constructor(port) {
        this.port = port;
        this.clients = new Map();
        this.server = null;
    }
    
    start() {
        this.server = net.createServer((socket) => {
            this.handleConnection(socket);
        });
        
        this.server.listen(this.port, () => {
            console.log(`Chat server listening on port ${this.port}`);
        });
    }
    
    handleConnection(socket) {
        const clientId = `${socket.remoteAddress}:${socket.remotePort}`;
        let username = null;
        
        socket.write('Enter your username: ');
        
        socket.on('data', (data) => {
            const message = data.toString().trim();
            
            if (!username) {
                username = message;
                this.clients.set(clientId, { socket, username });
                this.broadcast(`${username} joined the chat\n`, clientId);
                socket.write('Welcome to the chat!\n');
            } else {
                this.broadcast(`${username}: ${message}\n`, clientId);
            }
        });
        
        socket.on('end', () => {
            if (username) {
                this.broadcast(`${username} left the chat\n`, clientId);
                this.clients.delete(clientId);
            }
        });
        
        socket.on('error', (err) => {
            console.error(`Error for ${clientId}:`, err);
            this.clients.delete(clientId);
        });
    }
    
    broadcast(message, excludeId) {
        this.clients.forEach((client, id) => {
            if (id !== excludeId) {
                client.socket.write(message);
            }
        });
    }
}

const chatServer = new ChatServer(8080);
chatServer.start();
```

## Error Handling

### Robust Error Handling

```javascript
const net = require('net');

const server = net.createServer((socket) => {
    socket.setTimeout(30000); // 30 second timeout
    
    socket.on('timeout', () => {
        console.log('Socket timeout');
        socket.end('Connection timeout\n');
    });
    
    socket.on('error', (err) => {
        if (err.code === 'ECONNRESET') {
            console.log('Connection reset by client');
        } else if (err.code === 'EPIPE') {
            console.log('Broken pipe');
        } else {
            console.error('Socket error:', err);
        }
    });
    
    socket.on('data', (data) => {
        try {
            // Process data
            const message = data.toString();
            socket.write(`Processed: ${message}`);
        } catch (err) {
            console.error('Data processing error:', err);
            socket.write('ERROR: Invalid data\n');
        }
    });
});

server.on('error', (err) => {
    if (err.code === 'EADDRINUSE') {
        console.error('Port already in use');
        process.exit(1);
    } else {
        console.error('Server error:', err);
    }
});

server.listen(8080);
```

## Performance Optimization

### Connection Limits

```javascript
const net = require('net');

const MAX_CONNECTIONS = 100;
let activeConnections = 0;

const server = net.createServer((socket) => {
    if (activeConnections >= MAX_CONNECTIONS) {
        socket.end('Server is full. Try again later.\n');
        return;
    }
    
    activeConnections++;
    console.log(`Active connections: ${activeConnections}`);
    
    socket.on('end', () => {
        activeConnections--;
    });
    
    socket.on('error', () => {
        activeConnections--;
    });
    
    // Handle socket...
});

server.listen(8080);
```

### Backpressure Handling

```javascript
socket.on('data', (data) => {
    const canContinue = socket.write(data);
    
    if (!canContinue) {
        // Buffer is full, pause reading
        socket.pause();
        
        socket.once('drain', () => {
            // Buffer drained, resume reading
            socket.resume();
        });
    }
});
```

## SSL/TLS Support

### TLS Server

```javascript
const tls = require('tls');
const fs = require('fs');

const options = {
    key: fs.readFileSync('server-key.pem'),
    cert: fs.readFileSync('server-cert.pem')
};

const server = tls.createServer(options, (socket) => {
    console.log('Secure connection established');
    
    socket.write('Welcome to secure server\n');
    
    socket.on('data', (data) => {
        socket.write(`Secure echo: ${data}`);
    });
});

server.listen(8443, () => {
    console.log('TLS server listening on port 8443');
});
```

## Metrics and Monitoring

### Connection Statistics

```javascript
const net = require('net');

class MonitoredServer {
    constructor(port) {
        this.port = port;
        this.stats = {
            totalConnections: 0,
            activeConnections: 0,
            bytesReceived: 0,
            bytesSent: 0
        };
    }
    
    start() {
        const server = net.createServer((socket) => {
            this.stats.totalConnections++;
            this.stats.activeConnections++;
            
            socket.on('data', (data) => {
                this.stats.bytesReceived += data.length;
            });
            
            const originalWrite = socket.write.bind(socket);
            socket.write = (data, ...args) => {
                this.stats.bytesSent += data.length;
                return originalWrite(data, ...args);
            };
            
            socket.on('end', () => {
                this.stats.activeConnections--;
            });
        });
        
        server.listen(this.port);
        
        // Log stats every 10 seconds
        setInterval(() => {
            console.log('Server Stats:', this.stats);
        }, 10000);
    }
}

const server = new MonitoredServer(8080);
server.start();
```

## Best Practices

1. **Handle All Events**: data, end, error, timeout
2. **Set Timeouts**: Prevent hanging connections
3. **Limit Connections**: Protect against resource exhaustion
4. **Handle Backpressure**: Use pause/resume for flow control
5. **Validate Input**: Always validate incoming data
6. **Use TLS**: Encrypt sensitive communication
7. **Log Errors**: Comprehensive error logging
8. **Monitor Performance**: Track connections and throughput

## Kết luận

Node.js Net module cung cấp API mạnh mẽ để xây dựng TCP servers. Với event-driven architecture, Node.js rất phù hợp cho các ứng dụng network có nhiều concurrent connections.
