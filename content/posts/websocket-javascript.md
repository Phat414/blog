---
title: "WebSocket API trong JavaScript"
date: 2025-12-15
description: "Khám phá WebSocket để tạo kết nối real-time giữa client và server. Ví dụ thực tế về chat application..."
image: "https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=800&h=500&fit=crop"
---

WebSocket cung cấp kết nối hai chiều (full-duplex) giữa client và server, cho phép giao tiếp real-time hiệu quả hơn nhiều so với HTTP polling.

## WebSocket vs HTTP

### HTTP Polling
- Client liên tục gửi request để kiểm tra dữ liệu mới
- Tốn băng thông và tài nguyên server
- Độ trễ cao

### WebSocket
- Kết nối persistent hai chiều
- Server có thể push data đến client
- Độ trễ thấp, hiệu suất cao

## Creating WebSocket Connection

### Client-side JavaScript

```javascript
// Tạo WebSocket connection
const ws = new WebSocket('ws://localhost:8080');

// Connection opened
ws.addEventListener('open', (event) => {
    console.log('Connected to server');
    ws.send('Hello Server!');
});

// Listen for messages
ws.addEventListener('message', (event) => {
    console.log('Message from server:', event.data);
});

// Connection closed
ws.addEventListener('close', (event) => {
    console.log('Disconnected from server');
});

// Error handler
ws.addEventListener('error', (error) => {
    console.error('WebSocket error:', error);
});
```

## WebSocket Server (Node.js)

Sử dụng thư viện `ws`:

```javascript
const WebSocket = require('ws');

const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws) => {
    console.log('Client connected');
    
    // Receive message
    ws.on('message', (message) => {
        console.log('Received:', message);
        
        // Echo back to client
        ws.send(`Server received: ${message}`);
    });
    
    // Client disconnected
    ws.on('close', () => {
        console.log('Client disconnected');
    });
});
```

## Building a Chat Application

### Chat Client

```javascript
class ChatClient {
    constructor(url) {
        this.ws = new WebSocket(url);
        this.setupEventListeners();
    }
    
    setupEventListeners() {
        this.ws.onopen = () => {
            console.log('Connected to chat');
            this.updateStatus('Connected');
        };
        
        this.ws.onmessage = (event) => {
            const message = JSON.parse(event.data);
            this.displayMessage(message);
        };
        
        this.ws.onclose = () => {
            this.updateStatus('Disconnected');
        };
    }
    
    sendMessage(text) {
        const message = {
            type: 'message',
            text: text,
            timestamp: Date.now()
        };
        this.ws.send(JSON.stringify(message));
    }
    
    displayMessage(message) {
        const messageDiv = document.createElement('div');
        messageDiv.textContent = `${message.user}: ${message.text}`;
        document.getElementById('messages').appendChild(messageDiv);
    }
}

// Usage
const chat = new ChatClient('ws://localhost:8080');
```

## Broadcast Messages

Gửi message tới tất cả connected clients:

```javascript
wss.on('connection', (ws) => {
    ws.on('message', (data) => {
        const message = JSON.parse(data);
        
        // Broadcast to all clients
        wss.clients.forEach((client) => {
            if (client.readyState === WebSocket.OPEN) {
                client.send(JSON.stringify(message));
            }
        });
    });
});
```

## Connection Management

### Heartbeat/Ping-Pong

Giữ connection alive và phát hiện connection bị đứt:

```javascript
// Server-side
const interval = setInterval(() => {
    wss.clients.forEach((ws) => {
        if (ws.isAlive === false) {
            return ws.terminate();
        }
        ws.isAlive = false;
        ws.ping();
    });
}, 30000);

wss.on('connection', (ws) => {
    ws.isAlive = true;
    ws.on('pong', () => {
        ws.isAlive = true;
    });
});
```

## Reconnection Strategy

Tự động kết nối lại khi mất kết nối:

```javascript
class ReconnectingWebSocket {
    constructor(url) {
        this.url = url;
        this.reconnectInterval = 1000;
        this.maxReconnectInterval = 30000;
        this.connect();
    }
    
    connect() {
        this.ws = new WebSocket(this.url);
        
        this.ws.onopen = () => {
            console.log('Connected');
            this.reconnectInterval = 1000;
        };
        
        this.ws.onclose = () => {
            console.log('Disconnected, reconnecting...');
            setTimeout(() => {
                this.reconnectInterval = Math.min(
                    this.reconnectInterval * 2,
                    this.maxReconnectInterval
                );
                this.connect();
            }, this.reconnectInterval);
        };
    }
}
```

## Security Considerations

- Sử dụng WSS (WebSocket Secure) trong production
- Validate và sanitize tất cả messages
- Implement authentication và authorization
- Set rate limits để tránh abuse
- Handle connection limits

## Kết luận

WebSocket là công nghệ mạnh mẽ cho real-time applications. Hiểu rõ cách sử dụng WebSocket API sẽ giúp bạn xây dựng được các ứng dụng interactive như chat, live updates, gaming, và collaborative tools.
