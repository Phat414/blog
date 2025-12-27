---
title: "Xây dựng HTTP Server đơn giản bằng Java"
date: 2025-12-18
description: "Hướng dẫn chi tiết cách tạo một HTTP server từ đầu sử dụng Java Socket API, xử lý request và response..."
image: "https://images.unsplash.com/photo-1516259762381-22954d7d3ad2?w=800&h=500&fit=crop"
---

HTTP là giao thức nền tảng của World Wide Web. Trong bài viết này, chúng ta sẽ tìm hiểu cách xây dựng một HTTP server đơn giản từ đầu sử dụng Java Socket API.

## HTTP Protocol Basics

HTTP (Hypertext Transfer Protocol) là giao thức client-server hoạt động theo mô hình request-response. Client gửi HTTP request đến server, server xử lý và trả về HTTP response.

### HTTP Request Format

Một HTTP request bao gồm:
- Request line: Method (GET, POST,...) + Path + HTTP Version
- Headers: Thông tin bổ sung (Host, User-Agent, Content-Type,...)
- Body: Dữ liệu gửi lên (chỉ với POST, PUT,...)

## Tạo HTTP Server cơ bản

Chúng ta sẽ sử dụng ServerSocket để lắng nghe kết nối và xử lý HTTP requests.

### Simple HTTP Server

```java
public class SimpleHttpServer {
    private static final int PORT = 8080;
    
    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(PORT);
        System.out.println("Server started on port " + PORT);
        
        while (true) {
            Socket clientSocket = serverSocket.accept();
            handleClient(clientSocket);
        }
    }
    
    private static void handleClient(Socket socket) throws IOException {
        BufferedReader in = new BufferedReader(
            new InputStreamReader(socket.getInputStream()));
        PrintWriter out = new PrintWriter(socket.getOutputStream());
        
        // Đọc request line
        String requestLine = in.readLine();
        System.out.println("Request: " + requestLine);
        
        // Đọc headers
        String line;
        while ((line = in.readLine()) != null && !line.isEmpty()) {
            System.out.println("Header: " + line);
        }
        
        // Gửi HTTP response
        String response = "HTTP/1.1 200 OK\r\n" +
                         "Content-Type: text/html\r\n" +
                         "\r\n" +
                         "<html><body><h1>Hello!</h1></body></html>";
        
        out.print(response);
        out.flush();
        socket.close();
    }
}
```

## Xử lý các HTTP Methods

Để xử lý nhiều loại request khác nhau, chúng ta cần parse request line và xử lý theo method.

### Request Parser

```java
private static void handleRequest(Socket socket) throws IOException {
    BufferedReader in = new BufferedReader(
        new InputStreamReader(socket.getInputStream()));
    PrintWriter out = new PrintWriter(socket.getOutputStream());
    
    // Parse request line: GET /path HTTP/1.1
    String requestLine = in.readLine();
    String[] parts = requestLine.split(" ");
    String method = parts[0];
    String path = parts[1];
    
    // Xử lý theo method
    String response;
    if (method.equals("GET")) {
        response = handleGet(path);
    } else if (method.equals("POST")) {
        response = handlePost(path);
    } else {
        response = "HTTP/1.1 405 Method Not Allowed\r\n\r\n";
    }
    
    out.print(response);
    out.flush();
    socket.close();
}
```

## Multi-threaded Server

Để xử lý nhiều client đồng thời, chúng ta sử dụng thread pool:

```java
ExecutorService threadPool = Executors.newFixedThreadPool(10);

while (true) {
    Socket clientSocket = serverSocket.accept();
    threadPool.execute(() -> {
        try {
            handleClient(clientSocket);
        } catch (IOException e) {
            e.printStackTrace();
        }
    });
}
```

## Serving Static Files

Để serve file HTML, CSS, JS từ thư mục:

```java
private static String serveFile(String path) throws IOException {
    File file = new File("public" + path);
    
    if (!file.exists()) {
        return "HTTP/1.1 404 Not Found\r\n\r\nFile not found";
    }
    
    String contentType = getContentType(path);
    byte[] content = Files.readAllBytes(file.toPath());
    
    return "HTTP/1.1 200 OK\r\n" +
           "Content-Type: " + contentType + "\r\n" +
           "Content-Length: " + content.length + "\r\n\r\n";
}

private static String getContentType(String path) {
    if (path.endsWith(".html")) return "text/html";
    if (path.endsWith(".css")) return "text/css";
    if (path.endsWith(".js")) return "application/javascript";
    return "text/plain";
}
```

## Kết luận

Việc xây dựng HTTP server từ socket giúp bạn hiểu sâu về cách HTTP hoạt động. Tuy nhiên, trong thực tế, bạn nên sử dụng các framework như Spring Boot, Jetty, hoặc Tomcat cho production.

### Tính năng mở rộng

- Routing phức tạp với regex patterns
- Session management với cookies
- HTTPS support với SSL/TLS
- Compression (gzip) cho responses
- Caching và conditional requests
