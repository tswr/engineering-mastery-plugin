# C++ API Design Reference

C++ API design patterns for HTTP services and gRPC. For REST, Crow provides lightweight routing and JSON handling. For RPC, gRPC with Protocol Buffers provides strongly typed contracts and code generation. Protobuf schemas serve as the contract-first design artifact.

---

## Resource Modeling

```cpp
// Crow: lightweight REST framework with JSON support
#include "crow.h"

crow::SimpleApp app;

// GET — retrieval with proper 404 handling
CROW_ROUTE(app, "/users/<string>").methods(crow::HTTPMethod::GET)
([](const std::string& user_id) {
    auto user = user_repo.find(user_id);
    if (!user) {
        return crow::response(404, "application/problem+json",
            R"({"type":"/errors/not-found","title":"Not Found","status":404})");
    }
    crow::json::wvalue resp;
    resp["id"] = user->id;
    resp["name"] = user->name;
    resp["email"] = user->email;
    return crow::response(200, resp);
});

// POST — creation returns 201 with Location header
CROW_ROUTE(app, "/users").methods(crow::HTTPMethod::POST)
([](const crow::request& req) {
    auto body = crow::json::load(req.body);
    if (!body || !body.has("name") || !body.has("email")) {
        return crow::response(400, "application/problem+json",
            R"({"type":"/errors/validation","title":"Validation Error","status":400})");
    }
    auto user = user_repo.create(body["name"].s(), body["email"].s());
    crow::json::wvalue resp;
    resp["id"] = user.id;
    resp["name"] = user.name;
    auto r = crow::response(201, resp);
    r.add_header("Location", "/users/" + user.id);
    return r;
});
```

## Error Handling

```cpp
// Reusable RFC 7807 Problem Details helper
crow::response problem_response(int status, const std::string& type,
                                 const std::string& title,
                                 const std::string& detail) {
    crow::json::wvalue body;
    body["type"] = "/errors/" + type;
    body["title"] = title;
    body["status"] = status;
    body["detail"] = detail;
    return crow::response(status, "application/problem+json", body.dump());
}

CROW_ROUTE(app, "/orders/<string>").methods(crow::HTTPMethod::DELETE)
([](const std::string& order_id) {
    auto order = order_repo.find(order_id);
    if (!order)
        return problem_response(404, "not-found", "Not Found", "Order not found");
    if (order->status == OrderStatus::Shipped)
        return problem_response(409, "conflict", "Conflict", "Cannot delete shipped order");
    order_repo.remove(order_id);
    return crow::response(204);
});
```

## Documentation and Contracts (gRPC / Protobuf)

```protobuf
// user_service.proto — the contract-first design artifact
syntax = "proto3";
package userservice;

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
  rpc CreateUser(CreateUserRequest) returns (User);
}

message User { string id = 1; string name = 2; string email = 3; }
message GetUserRequest { string id = 1; }

message ListUsersRequest {
  int32 page_size = 1;       // server enforces a maximum
  string page_token = 2;     // opaque cursor from previous response
}
message ListUsersResponse {
  repeated User users = 1;
  string next_page_token = 2;
  int32 total_count = 3;
}

message CreateUserRequest {
  string name = 1; string email = 2;
  string idempotency_key = 3;  // for safe retries
}
```

```cpp
// gRPC C++ server — map domain errors to gRPC status codes
#include <grpcpp/grpcpp.h>
#include "user_service.grpc.pb.h"

class UserServiceImpl final : public userservice::UserService::Service {
    grpc::Status GetUser(grpc::ServerContext* context,
                         const userservice::GetUserRequest* request,
                         userservice::User* response) override {
        auto user = repo_.find(request->id());
        if (!user) {
            return grpc::Status(grpc::StatusCode::NOT_FOUND,
                                "User " + request->id() + " not found");
        }
        response->set_id(user->id);
        response->set_name(user->name);
        return grpc::Status::OK;
    }

    grpc::Status CreateUser(grpc::ServerContext* context,
                            const userservice::CreateUserRequest* request,
                            userservice::User* response) override {
        if (request->name().empty()) {
            return grpc::Status(grpc::StatusCode::INVALID_ARGUMENT, "name is required");
        }
        // Idempotency: return stored result if key was already processed
        if (!request->idempotency_key().empty()) {
            auto existing = idempotency_store_.get(request->idempotency_key());
            if (existing) { *response = *existing; return grpc::Status::OK; }
        }
        auto user = repo_.create(request->name(), request->email());
        response->set_id(user.id);
        response->set_name(user.name);
        if (!request->idempotency_key().empty())
            idempotency_store_.set(request->idempotency_key(), *response);
        return grpc::Status::OK;
    }

private:
    UserRepository repo_;
    IdempotencyStore idempotency_store_;
};
```

## Pagination

```cpp
grpc::Status ListUsers(grpc::ServerContext* context,
                       const userservice::ListUsersRequest* request,
                       userservice::ListUsersResponse* response) override {
    int page_size = std::clamp(request->page_size(), 1, 100);  // enforce max
    auto [users, total] = repo_.list(request->page_token(), page_size + 1);

    // Fetch one extra to detect if more results exist
    if (static_cast<int>(users.size()) > page_size) {
        users.pop_back();
        response->set_next_page_token(users.back().id);
    }
    response->set_total_count(total);
    for (const auto& u : users) {
        auto* user = response->add_users();
        user->set_id(u.id);
        user->set_name(u.name);
    }
    return grpc::Status::OK;
}
```
