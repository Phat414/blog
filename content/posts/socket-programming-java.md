---
title: "Socket Programming với Java"
date: 2025-12-20
description: "Tìm hiểu cách sử dụng Socket API trong Java để xây dựng ứng dụng client-server. Từ TCP socket đến UDP socket..."
image: "https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&h=500&fit=crop"
---

Socket programming là nền tảng cho mọi giao tiếp mạng trong Java. Trong bài viết này, chúng ta sẽ tìm hiểu cách sử dụng Socket API để xây dựng các ứng dụng client-server.

## Socket API Overview

Java cung cấp hai loại socket chính:
- **TCP Sockets**: Kết nối đáng tin cậy, có thứ tự (java.net.Socket)
- **UDP Sockets**: Kết nối không đảm bảo, nhanh (java.net.DatagramSocket)

### TCP Client-Server Model

TCP sử dụng mô hình client-server với kết nối đáng tin cậy.

## Creating a TCP Server

Để tạo TCP server, chúng ta sử dụng `ServerSocket`:

```java
public class SimpleServer {
    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(8080);
        System.out.println("Server started on port 8080");
        
        while (true) {
            Socket clientSocket = serverSocket.accept();
            System.out.println("Client connected: " + clientSocket.getInetAddress());
            
            // Handle client in new thread
            new Thread(() -> handleClient(clientSocket)).start();
        }
    }
    
    private static void handleClient(Socket socket) {
        try (
            BufferedReader in = new BufferedReader(
                new InputStreamReader(socket.getInputStream()));
            PrintWriter out = new PrintWriter(
                socket.getOutputStream(), true)
        ) {
            String message;
            while ((message = in.readLine()) != null) {
                System.out.println("Received: " + message);
                out.println("Echo: " + message);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## Creating a TCP Client

Client kết nối đến server và gửi/nhận dữ liệu:

```java
public class SimpleClient {
    public static void main(String[] args) {
        try (
            Socket socket = new Socket("localhost", 8080);
            PrintWriter out = new PrintWriter(
                socket.getOutputStream(), true);
            BufferedReader in = new BufferedReader(
                new InputStreamReader(socket.getInputStream()));
            BufferedReader stdIn = new BufferedReader(
                new InputStreamReader(System.in))
        ) {
            String userInput;
            while ((userInput = stdIn.readLine()) != null) {
                out.println(userInput);
                System.out.println("Server response: " + in.readLine());
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## UDP Socket Programming

UDP sockets cho phép gửi/nhận datagram packets mà không cần kết nối:

### UDP Server

```java
public class UDPServer {
    public static void main(String[] args) throws IOException {
        DatagramSocket socket = new DatagramSocket(9090);
        byte[] buffer = new byte[1024];
        
        System.out.println("UDP Server started on port 9090");
        
        while (true) {
            DatagramPacket packet = new DatagramPacket(buffer, buffer.length);
            socket.receive(packet);
            
            String message = new String(packet.getData(), 0, packet.getLength());
            System.out.println("Received: " + message);
            
            // Send response
            String response = "Echo: " + message;
            byte[] responseData = response.getBytes();
            DatagramPacket responsePacket = new DatagramPacket(
                responseData, responseData.length,
                packet.getAddress(), packet.getPort()
            );
            socket.send(responsePacket);
        }
    }
}
```

## Socket Options

Java Socket API cung cấp nhiều options để tùy chỉnh behavior:

```java
Socket socket = new Socket();
socket.setSoTimeout(5000);        // Read timeout
socket.setKeepAlive(true);        // Keep connection alive
socket.setTcpNoDelay(true);       // Disable Nagle's algorithm
socket.setReceiveBufferSize(8192); // Buffer size
```

## Error Handling

Xử lý lỗi đúng cách rất quan trọng:

- **SocketTimeoutException**: Timeout khi đọc dữ liệu
- **ConnectException**: Không thể kết nối đến server
- **IOException**: Lỗi I/O chung

## Best Practices

- Luôn close sockets trong `finally` block hoặc dùng try-with-resources
- Sử dụng thread pool thay vì tạo thread mới cho mỗi client
- Set timeout hợp lý để tránh blocking vô hạn
- Validate dữ liệu nhận được từ network
- Sử dụng buffer size phù hợp với use case

## Kết luận

Socket programming là nền tảng cho network programming trong Java. Hiểu rõ TCP và UDP sockets sẽ giúp bạn xây dựng được các ứng dụng mạng hiệu quả và đáng tin cậy.
