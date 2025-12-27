---
title: "Thiết kế RESTful API với Java Spring Boot"
date: 2025-12-12
description: "Best practices khi xây dựng RESTful API. Từ HTTP methods, status codes đến authentication và error handling..."
image: "https://images.unsplash.com/photo-1551033406-611cf9a28f67?w=800&h=500&fit=crop"
---

RESTful API là kiến trúc phổ biến nhất để xây dựng web services. Trong bài viết này, chúng ta sẽ tìm hiểu cách thiết kế và implement RESTful API với Java Spring Boot.

## REST Principles

REST (Representational State Transfer) dựa trên các nguyên tắc:
- **Stateless**: Mỗi request độc lập, không lưu trạng thái
- **Client-Server**: Tách biệt client và server
- **Cacheable**: Response có thể được cache
- **Uniform Interface**: Interface nhất quán
- **Layered System**: Hệ thống theo layer

## HTTP Methods

### CRUD Operations

- **GET**: Lấy dữ liệu (Read)
- **POST**: Tạo mới (Create)
- **PUT**: Cập nhật toàn bộ (Update)
- **PATCH**: Cập nhật một phần (Partial Update)
- **DELETE**: Xóa (Delete)

## Spring Boot REST Controller

### Basic Controller

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @Autowired
    private UserService userService;
    
    @GetMapping
    public List<User> getAllUsers() {
        return userService.findAll();
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<User> getUserById(@PathVariable Long id) {
        User user = userService.findById(id);
        if (user == null) {
            return ResponseEntity.notFound().build();
        }
        return ResponseEntity.ok(user);
    }
    
    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody @Valid User user) {
        User created = userService.create(user);
        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(created);
    }
    
    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(
            @PathVariable Long id,
            @RequestBody @Valid User user) {
        User updated = userService.update(id, user);
        return ResponseEntity.ok(updated);
    }
    
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

## HTTP Status Codes

### Common Status Codes

- **200 OK**: Request thành công
- **201 Created**: Resource được tạo thành công
- **204 No Content**: Thành công nhưng không có content
- **400 Bad Request**: Request không hợp lệ
- **401 Unauthorized**: Chưa authenticate
- **403 Forbidden**: Không có quyền truy cập
- **404 Not Found**: Resource không tồn tại
- **500 Internal Server Error**: Lỗi server

## Request/Response DTOs

### Data Transfer Objects

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class UserDTO {
    private Long id;
    
    @NotBlank(message = "Name is required")
    private String name;
    
    @Email(message = "Invalid email format")
    private String email;
    
    @Size(min = 8, message = "Password must be at least 8 characters")
    private String password;
}

@Data
public class UserResponseDTO {
    private Long id;
    private String name;
    private String email;
    private LocalDateTime createdAt;
}
```

## Exception Handling

### Global Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            ex.getMessage(),
            LocalDateTime.now()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ValidationErrorResponse> handleValidation(
            MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error -> 
            errors.put(error.getField(), error.getDefaultMessage())
        );
        
        ValidationErrorResponse response = new ValidationErrorResponse(
            HttpStatus.BAD_REQUEST.value(),
            "Validation failed",
            errors
        );
        return ResponseEntity.badRequest().body(response);
    }
}
```

## Pagination & Sorting

### Pageable Endpoints

```java
@GetMapping
public Page<UserDTO> getUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size,
        @RequestParam(defaultValue = "id") String sortBy,
        @RequestParam(defaultValue = "asc") String direction) {
    
    Sort sort = direction.equalsIgnoreCase("desc") 
        ? Sort.by(sortBy).descending() 
        : Sort.by(sortBy).ascending();
    
    Pageable pageable = PageRequest.of(page, size, sort);
    return userService.findAll(pageable);
}
```

## API Versioning

### URI Versioning

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {
    // Version 1 implementation
}

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {
    // Version 2 implementation
}
```

## Authentication & Authorization

### JWT Authentication

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .authorizeRequests()
                .antMatchers("/api/auth/**").permitAll()
                .antMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            .and()
            .addFilter(new JwtAuthenticationFilter(authenticationManager()))
            .addFilter(new JwtAuthorizationFilter(authenticationManager()))
            .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS);
    }
}
```

## API Documentation

### Swagger/OpenAPI

```java
@Configuration
@EnableSwagger2
public class SwaggerConfig {
    
    @Bean
    public Docket api() {
        return new Docket(DocumentationType.SWAGGER_2)
            .select()
            .apis(RequestHandlerSelectors.basePackage("com.example.api"))
            .paths(PathSelectors.any())
            .build()
            .apiInfo(apiInfo());
    }
    
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
            .title("User API")
            .description("RESTful API for User Management")
            .version("1.0.0")
            .build();
    }
}
```

## Best Practices

1. **Use Nouns for Resources**: `/api/users` not `/api/getUsers`
2. **Use HTTP Methods Correctly**: GET for read, POST for create, etc.
3. **Return Appropriate Status Codes**: 200, 201, 404, 500, etc.
4. **Version Your API**: `/api/v1/`, `/api/v2/`
5. **Validate Input**: Use `@Valid` and custom validators
6. **Handle Errors Gracefully**: Consistent error responses
7. **Use DTOs**: Separate domain models from API models
8. **Document Your API**: Swagger, OpenAPI, etc.
9. **Implement Pagination**: For large datasets
10. **Security First**: Authentication, authorization, HTTPS

## Kết luận

RESTful API design là kỹ năng quan trọng trong phát triển web services. Tuân thủ các best practices sẽ giúp API của bạn dễ sử dụng, maintain, và scale.
