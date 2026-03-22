# Python API Design Reference

Idiomatic Python patterns for API design using FastAPI. FastAPI provides automatic OpenAPI generation, dependency injection, and Pydantic-based validation out of the box.

---

## Resource Modeling

```python
from pydantic import BaseModel, Field
from uuid import UUID
from datetime import datetime
from fastapi import FastAPI, HTTPException, status

# Separate request/response models — never expose internal fields
class UserCreate(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    email: str = Field(..., pattern=r"^[^@\s]+@[^@\s]+\.[^@\s]+$")

class UserUpdate(BaseModel):
    """Partial update — all fields optional."""
    name: str | None = Field(None, min_length=1, max_length=100)
    email: str | None = None

class UserResponse(BaseModel):
    id: UUID
    name: str
    email: str
    created_at: datetime
    model_config = {"from_attributes": True}  # allows ORM object conversion

app = FastAPI()

@app.post("/users", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(user: UserCreate):
    # FastAPI validates request body against UserCreate automatically
    return await user_repo.create(user)

@app.get("/users/{user_id}", response_model=UserResponse)
async def get_user(user_id: UUID):
    user = await user_repo.get(user_id)
    if not user:
        raise HTTPException(status_code=404, detail="User not found")
    return user

@app.patch("/users/{user_id}", response_model=UserResponse)
async def update_user(user_id: UUID, updates: UserUpdate):
    # exclude_unset=True sends only fields the client actually provided
    return await user_repo.update(user_id, updates.model_dump(exclude_unset=True))
```

## Error Handling

```python
from fastapi import Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

# RFC 7807 Problem Details structure
class ProblemDetail(BaseModel):
    type: str = "about:blank"
    title: str
    status: int
    detail: str

class DomainError(Exception):
    def __init__(self, status: int, title: str, detail: str):
        self.status = status
        self.title = title
        self.detail = detail

@app.exception_handler(DomainError)
async def domain_error_handler(request: Request, exc: DomainError):
    return JSONResponse(
        status_code=exc.status,
        content=ProblemDetail(
            type=f"/errors/{exc.title.lower().replace(' ', '-')}",
            title=exc.title, status=exc.status, detail=exc.detail,
        ).model_dump(),
    )

# Return ALL validation errors in bulk, not just the first
@app.exception_handler(RequestValidationError)
async def validation_error_handler(request: Request, exc: RequestValidationError):
    errors = [
        {"field": ".".join(str(loc) for loc in e["loc"]), "message": e["msg"]}
        for e in exc.errors()
    ]
    return JSONResponse(status_code=422, content={
        "type": "/errors/validation", "title": "Validation Error",
        "status": 422, "detail": f"{len(errors)} validation error(s)",
        "errors": errors,
    })
```

## Pagination

```python
import base64

class PaginatedResponse(BaseModel):
    items: list[UserResponse]
    next_page_token: str | None = None
    total_count: int | None = None

@app.get("/users", response_model=PaginatedResponse)
async def list_users(page_token: str | None = None, page_size: int = 20):
    page_size = min(page_size, 100)  # enforce max server-side
    cursor = base64.urlsafe_b64decode(page_token).decode() if page_token else None
    users, total = await user_repo.list(cursor=cursor, limit=page_size + 1)

    # Fetch one extra to detect whether more results exist
    next_token = None
    if len(users) > page_size:
        users = users[:page_size]
        next_token = base64.urlsafe_b64encode(str(users[-1].id).encode()).decode()
    return PaginatedResponse(items=users, next_page_token=next_token, total_count=total)
```

## Idempotency

```python
from fastapi import Header

@app.post("/orders", response_model=OrderResponse, status_code=201)
async def create_order(
    order: OrderCreate,
    idempotency_key: str = Header(..., alias="Idempotency-Key"),
):
    existing = await idempotency_store.get(idempotency_key)
    if existing:
        return existing  # return stored result without re-processing
    result = await order_service.create(order)
    await idempotency_store.set(idempotency_key, result, ttl=86400)  # 24h TTL
    return result
```

## Documentation and Contracts

```python
# FastAPI generates OpenAPI 3.1 from type annotations and docstrings
app = FastAPI(title="User Service API", version="1.0.0")

@app.get("/users/{user_id}", response_model=UserResponse, tags=["users"],
         summary="Get a user by ID",
         responses={404: {"model": ProblemDetail, "description": "User not found"}})
async def get_user(user_id: UUID):
    """Retrieve a single user by their unique identifier."""
    ...

# Swagger UI at /docs, ReDoc at /redoc
# Export spec: import json; print(json.dumps(app.openapi(), indent=2))
```
