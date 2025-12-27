---
title: "SSL/TLS và HTTPS trong Java"
date: 2025-12-05
description: "Tìm hiểu cách implement SSL/TLS cho ứng dụng Java. Bảo mật kết nối mạng với certificates và encryption..."
image: "https://images.unsplash.com/photo-1563206767-5b18f218e8de?w=800&h=500&fit=crop"
---

SSL/TLS là giao thức bảo mật quan trọng nhất cho network communication. Trong bài viết này, chúng ta sẽ tìm hiểu cách implement SSL/TLS trong Java applications.

## SSL vs TLS

- **SSL (Secure Sockets Layer)**: Phiên bản cũ (đã deprecated)
- **TLS (Transport Layer Security)**: Phiên bản hiện đại và bảo mật hơn

TLS 1.2 và TLS 1.3 là các phiên bản được khuyến nghị sử dụng hiện nay.

## How TLS Works

### TLS Handshake Process

1. Client gửi "Client Hello" với supported cipher suites
2. Server trả về "Server Hello" và certificate
3. Client verify certificate
4. Trao đổi keys sử dụng asymmetric encryption
5. Establish encrypted connection với symmetric encryption

## Creating SSL/TLS Server

### HTTPS Server với Java

```java
import javax.net.ssl.*;
import java.io.*;
import java.security.*;

public class SecureServer {
    public static void main(String[] args) throws Exception {
        // Load keystore
        KeyStore keyStore = KeyStore.getInstance("JKS");
        FileInputStream keyStoreFile = new FileInputStream("server.jks");
        keyStore.load(keyStoreFile, "password".toCharArray());
        
        // Create key manager factory
        KeyManagerFactory kmf = KeyManagerFactory.getInstance("SunX509");
        kmf.init(keyStore, "password".toCharArray());
        
        // Create SSL context
        SSLContext sslContext = SSLContext.getInstance("TLS");
        sslContext.init(kmf.getKeyManagers(), null, null);
        
        // Create server socket
        SSLServerSocketFactory sslServerSocketFactory = 
            sslContext.getServerSocketFactory();
        SSLServerSocket serverSocket = 
            (SSLServerSocket) sslServerSocketFactory.createServerSocket(8443);
        
        System.out.println("Secure server started on port 8443");
        
        while (true) {
            SSLSocket clientSocket = (SSLSocket) serverSocket.accept();
            handleClient(clientSocket);
        }
    }
    
    private static void handleClient(SSLSocket socket) throws IOException {
        BufferedReader in = new BufferedReader(
            new InputStreamReader(socket.getInputStream()));
        PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
        
        String message = in.readLine();
        out.println("Secure echo: " + message);
        
        socket.close();
    }
}
```

## Creating SSL/TLS Client

### HTTPS Client

```java
public class SecureClient {
    public static void main(String[] args) throws Exception {
        // Trust all certificates (development only!)
        TrustManager[] trustAllCerts = new TrustManager[] {
            new X509TrustManager() {
                public X509Certificate[] getAcceptedIssuers() { return null; }
                public void checkClientTrusted(X509Certificate[] certs, String authType) {}
                public void checkServerTrusted(X509Certificate[] certs, String authType) {}
            }
        };
        
        SSLContext sslContext = SSLContext.getInstance("TLS");
        sslContext.init(null, trustAllCerts, new SecureRandom());
        
        SSLSocketFactory sslSocketFactory = sslContext.getSocketFactory();
        SSLSocket socket = (SSLSocket) sslSocketFactory.createSocket("localhost", 8443);
        
        PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
        BufferedReader in = new BufferedReader(
            new InputStreamReader(socket.getInputStream()));
        
        out.println("Hello Secure Server!");
        String response = in.readLine();
        System.out.println("Server response: " + response);
        
        socket.close();
    }
}
```

## Certificate Management

### Generating Self-Signed Certificate

```bash
# Generate keystore with self-signed certificate
keytool -genkeypair \
  -alias server \
  -keyalg RSA \
  -keysize 2048 \
  -validity 365 \
  -keystore server.jks \
  -storepass password \
  -keypass password \
  -dname "CN=localhost, OU=IT, O=MyCompany, L=City, ST=State, C=US"

# Export certificate
keytool -exportcert \
  -alias server \
  -keystore server.jks \
  -file server.cer \
  -storepass password

# Import to truststore
keytool -importcert \
  -alias server \
  -file server.cer \
  -keystore truststore.jks \
  -storepass password
```

## Spring Boot HTTPS Configuration

### application.properties

```properties
# HTTPS configuration
server.port=8443
server.ssl.enabled=true
server.ssl.key-store=classpath:keystore.jks
server.ssl.key-store-password=password
server.ssl.key-store-type=JKS
server.ssl.key-alias=server

# TLS version
server.ssl.enabled-protocols=TLSv1.2,TLSv1.3

# Cipher suites
server.ssl.ciphers=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
```

### Programmatic Configuration

```java
@Configuration
public class SSLConfig {
    
    @Bean
    public ServletWebServerFactory servletContainer() {
        TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory() {
            @Override
            protected void postProcessContext(Context context) {
                SecurityConstraint securityConstraint = new SecurityConstraint();
                securityConstraint.setUserConstraint("CONFIDENTIAL");
                SecurityCollection collection = new SecurityCollection();
                collection.addPattern("/*");
                securityConstraint.addCollection(collection);
                context.addConstraint(securityConstraint);
            }
        };
        
        tomcat.addAdditionalTomcatConnectors(redirectConnector());
        return tomcat;
    }
    
    private Connector redirectConnector() {
        Connector connector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
        connector.setScheme("http");
        connector.setPort(8080);
        connector.setSecure(false);
        connector.setRedirectPort(8443);
        return connector;
    }
}
```

## HTTP Client with SSL

### RestTemplate with SSL

```java
@Configuration
public class RestTemplateConfig {
    
    @Bean
    public RestTemplate restTemplate() throws Exception {
        TrustStrategy acceptingTrustStrategy = (X509Certificate[] chain, String authType) -> true;
        
        SSLContext sslContext = SSLContexts.custom()
            .loadTrustMaterial(null, acceptingTrustStrategy)
            .build();
        
        SSLConnectionSocketFactory csf = new SSLConnectionSocketFactory(sslContext);
        
        CloseableHttpClient httpClient = HttpClients.custom()
            .setSSLSocketFactory(csf)
            .build();
        
        HttpComponentsClientHttpRequestFactory requestFactory =
            new HttpComponentsClientHttpRequestFactory();
        requestFactory.setHttpClient(httpClient);
        
        return new RestTemplate(requestFactory);
    }
}
```

## Certificate Validation

### Custom Trust Manager

```java
public class CustomTrustManager implements X509TrustManager {
    
    private final X509TrustManager defaultTrustManager;
    
    public CustomTrustManager() throws Exception {
        TrustManagerFactory tmf = TrustManagerFactory.getInstance(
            TrustManagerFactory.getDefaultAlgorithm());
        tmf.init((KeyStore) null);
        
        for (TrustManager tm : tmf.getTrustManagers()) {
            if (tm instanceof X509TrustManager) {
                defaultTrustManager = (X509TrustManager) tm;
                return;
            }
        }
        throw new IllegalStateException("No X509TrustManager found");
    }
    
    @Override
    public void checkServerTrusted(X509Certificate[] chain, String authType) 
            throws CertificateException {
        // Custom validation logic
        if (chain.length == 0) {
            throw new CertificateException("Certificate chain is empty");
        }
        
        // Validate certificate dates
        for (X509Certificate cert : chain) {
            cert.checkValidity();
        }
        
        // Delegate to default trust manager
        defaultTrustManager.checkServerTrusted(chain, authType);
    }
    
    @Override
    public void checkClientTrusted(X509Certificate[] chain, String authType) 
            throws CertificateException {
        defaultTrustManager.checkClientTrusted(chain, authType);
    }
    
    @Override
    public X509Certificate[] getAcceptedIssuers() {
        return defaultTrustManager.getAcceptedIssuers();
    }
}
```

## Security Best Practices

1. **Use Strong Cipher Suites**: Chỉ enable các cipher suites mạnh
2. **Enable TLS 1.2+**: Disable SSLv3, TLS 1.0, TLS 1.1
3. **Validate Certificates**: Luôn validate server certificates
4. **Use Proper Key Sizes**: RSA 2048 bits minimum
5. **Certificate Pinning**: Pin certificates trong mobile/desktop apps
6. **Regular Updates**: Update certificates trước khi expire
7. **HSTS**: Enable HTTP Strict Transport Security
8. **Monitor Certificates**: Theo dõi expiration dates

## Common Issues

### Certificate Errors

- **sun.security.validator.ValidatorException**: Certificate chain không hợp lệ
- **javax.net.ssl.SSLHandshakeException**: Handshake  failed
- **CertificateExpiredException**: Certificate đã hết hạn

### Debugging SSL Issues

```bash
# Enable SSL debug logging
-Djavax.net.debug=ssl,handshake

# Check certificate
keytool -list -v -keystore keystore.jks
```

## Kết luận

SSL/TLS là nền tảng bảo mật cho mọi ứng dụng web và network. Hiểu rõ cách implement và configure SSL/TLS trong Java sẽ giúp bạn xây dựng được các ứng dụng an toàn và đáng tin cậy.
