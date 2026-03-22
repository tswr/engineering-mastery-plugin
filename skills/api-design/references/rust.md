# Rust API Design Reference

Idiomatic Rust patterns for API design using Axum. Axum builds on Tower for middleware (rate limiting, timeouts) and uses Serde for serialization. Error handling leverages the type system with `thiserror` for structured domain errors.

---

## Resource Modeling

```rust
use axum::{
    extract::{Path, Query, State},
    http::StatusCode,
    routing::{get, post, patch},
    Json, Router,
};
use serde::{Deserialize, Serialize};
use uuid::Uuid;

// Separate request/response types — never expose internal models
#[derive(Deserialize)]
struct CreateUser { name: String, email: String }

#[derive(Deserialize)]
struct UpdateUser { name: Option<String>, email: Option<String> }

#[derive(Serialize)]
struct UserResponse { id: Uuid, name: String, email: String, created_at: String }

fn user_routes() -> Router<AppState> {
    Router::new()
        .route("/users", get(list_users).post(create_user))
        .route("/users/{user_id}", get(get_user).patch(update_user))
}

async fn create_user(
    State(state): State<AppState>, Json(input): Json<CreateUser>,
) -> Result<(StatusCode, Json<UserResponse>), ApiError> {
    let user = state.user_repo.create(&input.name, &input.email).await?;
    Ok((StatusCode::CREATED, Json(user.into())))  // 201 for creation
}

async fn get_user(
    State(state): State<AppState>, Path(user_id): Path<Uuid>,
) -> Result<Json<UserResponse>, ApiError> {
    let user = state.user_repo.find(user_id).await?
        .ok_or(ApiError::NotFound("User not found".into()))?;
    Ok(Json(user.into()))
}
```

## Error Handling

```rust
use axum::response::{IntoResponse, Response};

#[derive(Serialize)]
struct ProblemDetail { r#type: String, title: String, status: u16, detail: String }

#[derive(Debug, thiserror::Error)]  // thiserror generates Display impls
enum ApiError {
    #[error("not found: {0}")]
    NotFound(String),
    #[error("validation failed: {0}")]
    Validation(String),
    #[error("conflict: {0}")]
    Conflict(String),
    #[error("internal error")]
    Internal(#[from] anyhow::Error),
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let (status, err_type, title) = match &self {
            ApiError::NotFound(_) => (StatusCode::NOT_FOUND, "not-found", "Not Found"),
            ApiError::Validation(_) => (StatusCode::BAD_REQUEST, "validation", "Validation Error"),
            ApiError::Conflict(_) => (StatusCode::CONFLICT, "conflict", "Conflict"),
            ApiError::Internal(_) => (StatusCode::INTERNAL_SERVER_ERROR, "internal", "Internal Error"),
        };
        // Never expose internal error details to the client
        let detail = match &self {
            ApiError::Internal(_) => "An unexpected error occurred".to_string(),
            other => other.to_string(),
        };
        let body = ProblemDetail {
            r#type: format!("/errors/{err_type}"), title: title.into(),
            status: status.as_u16(), detail,
        };
        (status, Json(body)).into_response()
    }
}
```

## Pagination

```rust
use base64::{engine::general_purpose::URL_SAFE_NO_PAD, Engine};

#[derive(Deserialize)]
struct PaginationParams { page_token: Option<String>, page_size: Option<usize> }

#[derive(Serialize)]
struct PaginatedResponse<T: Serialize> {
    items: Vec<T>, next_page_token: Option<String>, total_count: u64,
}

async fn list_users(
    State(state): State<AppState>, Query(params): Query<PaginationParams>,
) -> Result<Json<PaginatedResponse<UserResponse>>, ApiError> {
    let page_size = params.page_size.unwrap_or(20).min(100);  // enforce max
    let cursor = params.page_token.as_ref()
        .map(|t| URL_SAFE_NO_PAD.decode(t))
        .transpose()
        .map_err(|_| ApiError::Validation("invalid page_token".into()))?
        .map(|b| String::from_utf8_lossy(&b).to_string());

    let (mut users, total) = state.user_repo.list(cursor.as_deref(), page_size + 1).await?;

    // Fetch one extra to detect whether more results exist
    let next_token = if users.len() > page_size {
        users.truncate(page_size);
        Some(URL_SAFE_NO_PAD.encode(users.last().unwrap().id.to_string()))
    } else { None };

    Ok(Json(PaginatedResponse {
        items: users.into_iter().map(Into::into).collect(),
        next_page_token: next_token, total_count: total,
    }))
}
```

## Idempotency

```rust
use axum::http::HeaderMap;

async fn create_order(
    State(state): State<AppState>, headers: HeaderMap, Json(input): Json<CreateOrder>,
) -> Result<(StatusCode, Json<OrderResponse>), ApiError> {
    let key = headers.get("idempotency-key")
        .and_then(|v| v.to_str().ok())
        .ok_or(ApiError::Validation("Idempotency-Key header is required".into()))?;

    if let Some(existing) = state.idempotency_store.get(key).await? {
        return Ok((StatusCode::OK, Json(existing)));  // return stored result
    }
    let order = state.order_service.create(input).await?;
    state.idempotency_store.set(key, &order, 86400).await?;  // 24h TTL
    Ok((StatusCode::CREATED, Json(order)))
}
```

## Rate Limiting

```rust
use tower::ServiceBuilder;
use tower_governor::{GovernorLayer, GovernorConfigBuilder};

fn app() -> Router<AppState> {
    // 60 requests per minute per IP, with burst allowance
    let governor_conf = GovernorConfigBuilder::default()
        .per_second(1).burst_size(60)
        .finish().expect("valid governor config");

    // tower_governor returns 429 with Retry-After header automatically
    Router::new()
        .merge(user_routes())
        .layer(ServiceBuilder::new().layer(GovernorLayer { config: governor_conf.into() }))
}
```
