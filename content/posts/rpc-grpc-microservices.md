---
title: "RPC và gRPC trong Microservices"
date: 2025-11-25
description: "Tìm hiểu Remote Procedure Call và gRPC framework. Implement service-to-service communication trong kiến trúc microservices..."
image: "https://images.unsplash.com/photo-1451187580459-43490279c0fa?w=800&h=500&fit=crop"
---

RPC (Remote Procedure Call) cho phép gọi hàm trên remote server như thể gọi hàm local. gRPC là RPC framework hiện đại từ Google, sử dụng HTTP/2 và Protocol Buffers.

## RPC Concepts

### Traditional RPC vs REST

**RPC:**
- Gọi hàm remote như local function
- Strongly typed với code generation
- Binary protocols (hiệu quả hơn)

**REST:**
- Resource-oriented
- HTTP methods (GET, POST, PUT, DELETE)
- Text-based (JSON, XML)

## gRPC Overview

### Key Features

- **HTTP/2**: Multiplexing, server push, header compression
- **Protocol Buffers**: Efficient binary serialization
- **Streaming**: Client, server, và bidirectional streaming
- **Code Generation**: Tự động generate client/server code
- **Multi-language**: Support nhiều ngôn ngữ

## Protocol Buffers

### Defining Messages

```protobuf
syntax = "proto3";

package user;

service UserService {
    rpc GetUser(GetUserRequest) returns (User);
    rpc ListUsers(ListUsersRequest) returns (stream User);
    rpc CreateUser(CreateUserRequest) returns (User);
    rpc UpdateUser(UpdateUserRequest) returns (User);
    rpc DeleteUser(DeleteUserRequest) returns (Empty);
}

message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
    int32 age = 4;
}

message GetUserRequest {
    int64 id = 1;
}

message ListUsersRequest {
    int32 page = 1;
    int32 page_size = 2;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
    int32 age = 3;
}

message UpdateUserRequest {
    int64 id = 1;
    string name = 2;
    string email = 3;
    int32 age = 4;
}

message DeleteUserRequest {
    int64 id = 1;
}

message Empty {}
```

## gRPC Server Implementation

### Java Server

```java
public class UserServiceImpl extends UserServiceGrpc.UserServiceImplBase {
    
    @Override
    public void getUser(GetUserRequest request, 
                       StreamObserver<User> responseObserver) {
        long userId = request.getId();
        
        // Fetch user from database
        User user = userRepository.findById(userId);
        
        if (user != null) {
            responseObserver.onNext(user);
            responseObserver.onCompleted();
        } else {
            responseObserver.onError(
                Status.NOT_FOUND
                    .withDescription("User not found")
                    .asRuntimeException()
            );
        }
    }
    
    @Override
    public void listUsers(ListUsersRequest request,
                         StreamObserver<User> responseObserver) {
        int page = request.getPage();
        int pageSize = request.getPageSize();
        
        List<User> users = userRepository.findAll(page, pageSize);
        
        for (User user : users) {
            responseObserver.onNext(user);
        }
        
        responseObserver.onCompleted();
    }
    
    @Override
    public void createUser(CreateUserRequest request,
                          StreamObserver<User> responseObserver) {
        User user = User.newBuilder()
            .setName(request.getName())
            .setEmail(request.getEmail())
            .setAge(request.getAge())
            .build();
        
        User created = userRepository.save(user);
        
        responseObserver.onNext(created);
        responseObserver.onCompleted();
    }
}

// Start server
public class GrpcServer {
    public static void main(String[] args) throws IOException, InterruptedException {
        Server server = ServerBuilder.forPort(9090)
            .addService(new UserServiceImpl())
            .build()
            .start();
        
        System.out.println("Server started on port 9090");
        server.awaitTermination();
    }
}
```

## gRPC Client

### Java Client

```java
public class UserServiceClient {
    private final UserServiceGrpc.UserServiceBlockingStub blockingStub;
    private final UserServiceGrpc.UserServiceStub asyncStub;
    
    public UserServiceClient(Channel channel) {
        this.blockingStub = UserServiceGrpc.newBlockingStub(channel);
        this.asyncStub = UserServiceGrpc.newStub(channel);
    }
    
    public User getUser(long id) {
        GetUserRequest request = GetUserRequest.newBuilder()
            .setId(id)
            .build();
        
        try {
            return blockingStub.getUser(request);
        } catch (StatusRuntimeException e) {
            logger.log(Level.WARNING, "RPC failed: {0}", e.getStatus());
            return null;
        }
    }
    
    public void listUsers(int page, int pageSize) {
        ListUsersRequest request = ListUsersRequest.newBuilder()
            .setPage(page)
            .setPageSize(pageSize)
            .build();
        
        Iterator<User> users = blockingStub.listUsers(request);
        
        while (users.hasNext()) {
            User user = users.next();
            System.out.println("User: " + user.getName());
        }
    }
    
    public static void main(String[] args) {
        ManagedChannel channel = ManagedChannelBuilder
            .forAddress("localhost", 9090)
            .usePlaintext()
            .build();
        
        UserServiceClient client = new UserServiceClient(channel);
        
        User user = client.getUser(1);
        System.out.println("Found user: " + user);
        
channel.shutdown();
    }
}
```

## Streaming RPCs

### Server Streaming

```java
@Override
public void streamUsers(StreamRequest request,
                       StreamObserver<User> responseObserver) {
    userRepository.findAll().forEach(user -> {
        responseObserver.onNext(user);
        
        try {
            Thread.sleep(100); // Simulate delay
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    });
    
    responseObserver.onCompleted();
}
```

### Client Streaming

```java
@Override
public StreamObserver<CreateUserRequest> batchCreateUsers(
        StreamObserver<BatchCreateResponse> responseObserver) {
    
    return new StreamObserver<CreateUserRequest>() {
        private int count = 0;
        
        @Override
        public void onNext(CreateUserRequest request) {
            User user = createUser(request);
            count++;
        }
        
        @Override
        public void onError(Throwable t) {
            logger.log(Level.WARNING, "Error: {0}", t);
        }
        
        @Override
        public void onCompleted() {
            BatchCreateResponse response = BatchCreateResponse.newBuilder()
                .setCount(count)
                .build();
            
            responseObserver.onNext(response);
            responseObserver.onCompleted();
        }
    };
}
```

### Bidirectional Streaming

```java
@Override
public StreamObserver<ChatMessage> chat(
        StreamObserver<ChatMessage> responseObserver) {
    
    return new StreamObserver<ChatMessage>() {
        @Override
        public void onNext(ChatMessage message) {
            // Broadcast to all connected clients
            broadcastMessage(message, responseObserver);
        }
        
        @Override
        public void onError(Throwable t) {
            logger.log(Level.WARNING, "Chat error: {0}", t);
        }
        
        @Override
        public void onCompleted() {
            responseObserver.onCompleted();
        }
    };
}
```

## Error Handling

### gRPC Status Codes

```java
public class ErrorHandler {
    public static StatusRuntimeException notFound(String message) {
        return Status.NOT_FOUND
            .withDescription(message)
            .asRuntimeException();
    }
    
    public static StatusRuntimeException invalidArgument(String message) {
        return Status.INVALID_ARGUMENT
            .withDescription(message)
            .asRuntimeException();
    }
    
    public static StatusRuntimeException internal(String message) {
        return Status.INTERNAL
            .withDescription(message)
            .asRuntimeException();
    }
}

// Usage
if (user == null) {
    throw ErrorHandler.notFound("User not found");
}
```

## Interceptors

### Server Interceptor

```java
public class LoggingInterceptor implements ServerInterceptor {
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {
        
        logger.info("Method: " + call.getMethodDescriptor().getFullMethodName());
        logger.info("Headers: " + headers);
        
        return next.startCall(call, headers);
    }
}

// Add to server
Server server = ServerBuilder.forPort(9090)
    .addService(new UserServiceImpl())
    .intercept(new LoggingInterceptor())
    .build();
```

### Client Interceptor

```java
public class AuthInterceptor implements ClientInterceptor {
    private final String token;
    
    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(
            MethodDescriptor<ReqT, RespT> method,
            CallOptions callOptions,
            Channel next) {
        
        return new ForwardingClientCall.SimpleForwardingClientCall<ReqT, RespT>(
                next.newCall(method, callOptions)) {
            
            @Override
            public void start(Listener<RespT> responseListener, Metadata headers) {
                headers.put(Metadata.Key.of("authorization", ASCII_STRING_MARSHALLER),
                           "Bearer " + token);
                super.start(responseListener, headers);
            }
        };
    }
}
```

## Load Balancing

### Client-Side Load Balancing

```java
ManagedChannel channel = ManagedChannelBuilder
    .forTarget("dns:///my-service:9090")
    .defaultLoadBalancingPolicy("round_robin")
    .usePlaintext()
    .build();
```

## Security

### TLS/SSL

```java
// Server with TLS
Server server = NettyServerBuilder.forPort(9090)
    .addService(new UserServiceImpl())
    .useTransportSecurity(certChainFile, privateKeyFile)
    .build();

// Client with TLS
ManagedChannel channel = NettyChannelBuilder
    .forAddress("localhost", 9090)
    .sslContext(GrpcSslContexts.forClient()
        .trustManager(trustCertCollectionFile)
        .build())
    .build();
```

## Spring Boot Integration

### gRPC Server with Spring Boot

```java
@GrpcService
public class UserGrpcService extends UserServiceGrpc.UserServiceImplBase {
    
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public void getUser(GetUserRequest request,
                       StreamObserver<User> responseObserver) {
        // Implementation
    }
}

// application.yml
grpc:
  server:
    port: 9090
```

## Best Practices

1. **Use Protocol Buffers**: Efficient serialization
2. **Implement Timeouts**: Prevent hanging calls
3. **Handle Errors Properly**: Use appropriate status codes
4. **Enable Compression**: Reduce network usage
5. **Use Interceptors**: Logging, authentication, metrics
6. **Connection Pooling**: Reuse connections
7. **Health Checks**: Monitor service health
8. **Versioning**: Plan for API evolution
9. **Load Balancing**: Distribute load across instances
10. **TLS**: Encrypt communication in production

## Kết luận

gRPC là framework mạnh mẽ cho service-to-service communication trong microservices. Với HTTP/2, Protocol Buffers, và streaming support, gRPC cung cấp hiệu suất và tính năng vượt trội so với REST cho internal APIs.
