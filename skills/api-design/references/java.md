# Java API Design Reference

Idiomatic Java patterns for API design using Spring Boot. Spring provides annotation-driven routing, Jakarta Bean Validation for request validation, and springdoc-openapi for automatic OpenAPI generation. Error handling uses `@RestControllerAdvice` for consistent, centralized error responses.

---

## Resource Modeling

```java
// Request DTO — Jakarta Bean Validation enforces constraints
public record CreateUserRequest(
    @NotBlank @Size(max = 100) String name,
    @NotBlank @Email String email
) {}

public record UpdateUserRequest(
    @Size(max = 100) String name,  // nullable = partial update
    @Email String email
) {}

// Response DTO — separate from entity, includes server-generated fields
public record UserResponse(UUID id, String name, String email, Instant createdAt) {
    public static UserResponse from(User entity) {
        return new UserResponse(entity.getId(), entity.getName(),
                                entity.getEmail(), entity.getCreatedAt());
    }
}

@RestController
@RequestMapping("/users")
public class UserController {
    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)  // 201 for resource creation
    public UserResponse createUser(@Valid @RequestBody CreateUserRequest request) {
        return UserResponse.from(userService.create(request));
    }

    @GetMapping("/{userId}")
    public UserResponse getUser(@PathVariable UUID userId) {
        return userService.findById(userId)
            .map(UserResponse::from)
            .orElseThrow(() -> new ResourceNotFoundException("User", userId));
    }

    @PatchMapping("/{userId}")
    public UserResponse updateUser(@PathVariable UUID userId,
                                   @Valid @RequestBody UpdateUserRequest request) {
        return UserResponse.from(userService.update(userId, request));
    }

    @DeleteMapping("/{userId}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteUser(@PathVariable UUID userId) {
        userService.delete(userId);
    }
}
```

## Error Handling

```java
// Centralized error handling — Spring 6 ProblemDetail follows RFC 7807
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        var problem = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Not Found");
        problem.setType(URI.create("/errors/not-found"));
        return problem;  // serialized as application/problem+json
    }

    // Return ALL validation errors in bulk
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problem.setTitle("Validation Error");
        problem.setType(URI.create("/errors/validation"));
        List<Map<String, String>> errors = ex.getBindingResult().getFieldErrors()
            .stream()
            .map(fe -> Map.of("field", fe.getField(),
                              "message", Objects.requireNonNullElse(fe.getDefaultMessage(), "")))
            .toList();
        problem.setProperty("errors", errors);
        return problem;
    }

    // Catch-all: log details server-side, never expose stack traces
    @ExceptionHandler(Exception.class)
    public ProblemDetail handleUnexpected(Exception ex) {
        log.error("Unexpected error", ex);
        return ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR, "An unexpected error occurred");
    }
}
```

## Pagination

```java
public record PaginatedResponse<T>(List<T> items, String nextPageToken, long totalCount) {}

@GetMapping
public PaginatedResponse<UserResponse> listUsers(
        @RequestParam(required = false) String pageToken,
        @RequestParam(defaultValue = "20") @Max(100) int pageSize) {
    UUID cursor = (pageToken != null)
        ? UUID.fromString(new String(Base64.getUrlDecoder().decode(pageToken)))
        : null;

    List<User> users = userService.list(cursor, pageSize + 1);
    long total = userService.count();

    // Fetch one extra to detect whether more results exist
    String nextToken = null;
    if (users.size() > pageSize) {
        users = users.subList(0, pageSize);
        nextToken = Base64.getUrlEncoder().withoutPadding()
            .encodeToString(users.get(users.size() - 1).getId().toString().getBytes());
    }
    return new PaginatedResponse<>(
        users.stream().map(UserResponse::from).toList(), nextToken, total);
}
```

## Idempotency

```java
@PostMapping("/orders")
@ResponseStatus(HttpStatus.CREATED)
public OrderResponse createOrder(
        @Valid @RequestBody CreateOrderRequest request,
        @RequestHeader("Idempotency-Key") String idempotencyKey) {
    return idempotencyStore.get(idempotencyKey)
        .orElseGet(() -> {
            OrderResponse result = OrderResponse.from(orderService.create(request));
            idempotencyStore.set(idempotencyKey, result, Duration.ofHours(24));
            return result;
        });
}
```

## Documentation and Contracts

```java
// springdoc-openapi generates OpenAPI 3 from annotations and types
// Dependency: org.springdoc:springdoc-openapi-starter-webmvc-ui

@OpenAPIDefinition(info = @Info(title = "User Service API", version = "1.0.0"))
@SpringBootApplication
public class Application { }

@Operation(summary = "Get a user by ID",
    responses = {
        @ApiResponse(responseCode = "200", description = "User found"),
        @ApiResponse(responseCode = "404", description = "User not found",
            content = @Content(schema = @Schema(implementation = ProblemDetail.class)))
    })
@GetMapping("/{userId}")
public UserResponse getUser(@PathVariable UUID userId) { ... }

// Swagger UI at /swagger-ui.html, spec at /v3/api-docs
```
