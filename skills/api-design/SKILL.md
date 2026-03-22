---
name: api-design
description: "Apply when designing, reviewing, or evolving HTTP APIs, gRPC services, or library interfaces: resource modeling, naming, versioning, error handling, pagination, and backward compatibility. Trigger on mentions of REST, API design, endpoints, backward compatibility, API versioning, or contract-first design. Language-specific idioms are in references/."
---

# API Design Skill

Apply these principles whenever you design, review, or evolve an API — whether it is a RESTful HTTP service, a gRPC interface, or a public library surface. A good API is a contract between producer and consumer. Breaking that contract breaks trust. Designing it carelessly creates years of maintenance burden.

The overarching standard: **an API is a product.** Its consumers are its users, and their experience determines its success. Every decision — naming, error format, pagination strategy, versioning scheme — should be made from the consumer's perspective.

> **Language-specific API idioms** (C++, Python, Rust, Java) are in `references/`. This document covers language-agnostic principles.

---

## 1. Contract-First Design

Design the API contract before writing any implementation code. The contract — an OpenAPI specification, a protobuf schema, a GraphQL SDL — is the artifact that consumers depend on. Starting from the database schema or internal domain model produces APIs that leak implementation details and are painful to consume.

**Write the interface definition first.** Draft the OpenAPI spec or protobuf file, circulate it to consumers and stakeholders, collect feedback, and iterate on the contract before committing to implementation. This is the approach advocated by Geewax and by Jin, Sahni, and Shevat: the contract is the design document. Reviewing a spec is far cheaper than rewriting an implementation.

**Design for the consumer, not the implementation.** The resources and operations your API exposes should reflect what consumers need to accomplish, not how your backend is structured. If your database has a `user_account_records` table, your API should still expose `/users` — the internal name is irrelevant to the consumer. If the consumer needs a combined view of data from three microservices, the API should present that as a single resource, not force the consumer to orchestrate three calls.

**Make the interface deep.** From Ousterhout: the best interfaces are simple surfaces that hide significant complexity. An API endpoint that requires the caller to understand five internal subsystems to use it correctly is a shallow interface. A well-designed endpoint accepts a clear input, does substantial work internally, and returns a clear output. The ratio of functionality to interface complexity should be high.

A deep interface reduces the cognitive load on every consumer and concentrates complexity where it can be managed — on the server side. Shallow interfaces push complexity outward, forcing every consumer to reimplement orchestration logic that belongs on the server.

**Treat the spec as the source of truth.** Generate server stubs, client SDKs, and documentation from the specification. Never let the implementation diverge from the contract — if you change the code, update the spec first. Automated conformance tests should verify that the running service matches the published specification.

**Validate the contract with real use cases.** Before finalizing the spec, walk through the three to five most important consumer workflows end to end. Can the consumer accomplish each workflow with the resources and operations defined? Are there unnecessary round trips? Does the consumer need to know about internal details that should be hidden? If any workflow is awkward, the contract needs revision.

---

## 2. Resource Modeling and Naming

Resources are the fundamental concept of a REST API. Every entity the API exposes — users, orders, invoices — is a resource with a stable identity and a URI.

**Use nouns, not verbs, for resource names.** The HTTP method supplies the verb. Use `/users`, not `/getUsers` or `/createUser`. The Google API Design Guide is explicit on this point: resource names are nouns, and the standard methods (List, Get, Create, Update, Delete) map to HTTP verbs.

**Use consistent pluralization.** Collection resources are plural: `/users`, `/orders`, `/products`. A specific resource within the collection uses an identifier: `/users/{user_id}`. From Richardson and Amundsen: consistency in pluralization eliminates an entire category of guesswork for consumers.

**Model containment with hierarchical URIs.** When one resource logically belongs to another, express that relationship in the path: `/users/{user_id}/orders/{order_id}`. This communicates scope and access control naturally. Avoid nesting more than two or three levels deep — flatten the hierarchy if it becomes unwieldy. From the Google API Design Guide: deeply nested URIs are a sign that the resource model needs rethinking.

**Use stable, opaque identifiers.** Resource identifiers should be stable across the lifetime of the resource and should not encode internal details. UUIDs or server-generated IDs are preferable to sequential integers (which leak information about creation order and volume) or natural keys (which may change). From Geewax: the identifier is part of the API contract — if it changes, every consumer's stored references break.

**Name resources clearly and unambiguously.** From Geewax: avoid abbreviations, internal jargon, or names that could refer to multiple things. If the resource is an invoice, call it `/invoices`, not `/inv` or `/billing-documents`. Use kebab-case for multi-word path segments (`/line-items`), and camelCase or snake_case for field names — pick one convention and enforce it across the entire API.

**Use standard methods for standard operations.** Map CRUD operations to HTTP methods consistently: GET for retrieval, POST for creation, PUT or PATCH for updates, DELETE for removal. Reserve custom methods for operations that genuinely do not map to CRUD — and name them clearly when you do. From the Google API Design Guide: custom methods use a colon separator (e.g., `/users/{id}:deactivate`) to distinguish them from sub-resources.

**Distinguish between PUT and PATCH.** PUT replaces the entire resource — the client sends the complete representation, and any omitted fields are reset to defaults or removed. PATCH applies a partial update — only the fields included in the request are modified. From the Microsoft REST API Guidelines: PATCH is generally preferred for updates because it avoids the "read-modify-write" cycle that PUT requires and reduces the risk of accidentally overwriting concurrent changes.

---

## 3. Backward Compatibility

An API is a published contract. Once consumers depend on it, breaking changes impose real costs — broken builds, failed deployments, lost trust. Treat backward compatibility as a design constraint, not an afterthought.

**Backward-compatible changes are additive.** From Geewax: you can safely add new optional fields to responses, add new endpoints, add new optional query parameters, and add new enum values (if consumers handle unknown values). These changes do not break existing consumers.

**Breaking changes violate the contract.** The following are all breaking changes: removing a field, renaming a field, changing a field's type, changing a field from optional to required, changing the semantic meaning of a field, removing an endpoint, or changing a URL structure. Any of these will break existing consumers. Treat every one as a major version change.

**Identify breaking changes before they ship.** From Geewax: maintain automated compatibility checks that compare the current spec against the previous version. Flag any removed fields, type changes, or new required parameters before they reach production. Breaking changes discovered after deployment are orders of magnitude more expensive than those caught in review.

**Choose a versioning strategy and stick with it.** URL path versioning (`/v1/users`, `/v2/users`) is the most visible and widely adopted. Header versioning (via `Accept` or a custom header) keeps URIs clean but is less discoverable. From the Google API Design Guide: whichever strategy you choose, apply it consistently and document it clearly.

**Deprecate gracefully.** From the Google API Design Guide: announce deprecation well in advance. Provide a migration guide that explains what changed and how to update. Maintain the old version for a defined period — long enough for consumers to migrate. Mark deprecated fields or endpoints in the specification and return deprecation warnings in response headers.

**Communicate a deprecation timeline.** From the Google API Design Guide: a deprecation policy should include the announcement date, the date the deprecated version enters maintenance-only mode (no new features, only security fixes), and the date the deprecated version will be shut down. Consumers need a concrete timeline to plan their migration, not a vague "will be removed in a future version."

**Never redefine an existing field.** If `/v1/users` returns `age` as an integer, you cannot change it to a string in the same version. If the meaning or type of a field must change, introduce a new field with a distinct name and deprecate the old one. Consumers should never encounter a field whose type or meaning has silently changed.

**Design for forward compatibility.** Consumers should be written to tolerate unknown fields and unknown enum values. From Geewax: if a client fails on encountering a new field it does not recognize, that client is brittle. Encourage consumers to ignore unknown fields rather than rejecting them. This gives you room to evolve the API without breaking well-behaved clients.

---

## 4. Error Handling and Status Codes

Errors are part of the API contract. A well-designed error response tells the consumer what went wrong, why, and what they can do about it.

**Use HTTP status codes correctly.** From RFC 7231: status codes have defined semantics. A 200 means the request succeeded. A 404 means the resource was not found. A 400 means the client sent a malformed request. Returning 200 with an error message in the body violates the protocol and breaks clients that rely on status codes for control flow. Use the right code for the situation.

**Adopt a standard error response format.** RFC 7807 (Problem Details for HTTP APIs) defines a structured format with these fields:

- `type` — a URI identifying the error category
- `title` — a short, human-readable summary
- `status` — the HTTP status code
- `detail` — a human-readable explanation of this specific occurrence
- `instance` — a URI identifying this specific occurrence

Using a standard format means consumers can write generic error-handling code that works across every endpoint. The `type` URI allows consumers to programmatically distinguish between different error categories without parsing human-readable strings.

**Make errors actionable.** From the Google API Design Guide: an error message should tell the caller what went wrong and what they can do about it. "Invalid request" is useless. "The `email` field must be a valid email address; received `not-an-email`" is actionable. Include which field failed validation, what the constraint is, and what value was received.

**Map domain errors to HTTP status codes consistently.** Define a clear mapping from your domain's error categories to HTTP status codes and enforce it across the entire API. Validation errors are 400. Authentication failures are 401. Authorization failures are 403. Resource not found is 404. Conflicts (e.g., duplicate creation) are 409. Unprocessable content (valid syntax but invalid semantics) is 422. Internal errors are 500. Document this mapping and do not deviate from it.

**Never expose internal details in error responses.** Stack traces, database query errors, and internal service names do not belong in API error responses. They leak implementation details and create security risks. Log them server-side; return a safe, structured error to the consumer.

**Return validation errors in bulk.** When a request has multiple validation failures, return all of them in a single response rather than failing on the first one. This lets the consumer fix all issues in one pass rather than playing whack-a-mole with sequential requests.

**Distinguish between client errors and server errors.** From RFC 7231: 4xx status codes indicate that the client sent something wrong — the client should modify the request before retrying. 5xx status codes indicate that the server failed — the client may retry the same request. This distinction matters for automated retry logic. A client that retries a 400 indefinitely wastes resources; a client that gives up on a 503 misses a transient failure.

---

## 5. Pagination, Filtering, and Partial Responses

Any endpoint that returns a collection will eventually return too many items. Design for this from the start, not after the first production outage.

**Use cursor-based pagination for large or changing datasets.** From Geewax: cursor-based pagination uses an opaque token that points to a position in the result set. The server returns a `next_page_token`; the client sends it back to get the next page. This approach is stable even when items are inserted or deleted between requests, because the cursor encodes a position rather than an offset. The token should be opaque to the client — encoding implementation details into the cursor couples the client to your storage layer and prevents you from changing the underlying implementation.

**Set and enforce a maximum page size.** From the Google API Design Guide: always define a maximum page size and enforce it server-side. If a client requests 10,000 results per page, cap it at your defined maximum and return that maximum. Document the default page size and the maximum. Unbounded result sets are a denial-of-service vector.

**Use offset-based pagination only for small, stable datasets.** Offset pagination (`?page=3&page_size=20`) is simpler to implement and allows random access to pages, but it breaks when the underlying data changes between requests — items can be skipped or duplicated. Reserve it for admin dashboards and similar low-stakes use cases.

**Follow the page_token pattern.** From the Google API Design Guide: the List response includes a `next_page_token` field. The client sends it as `page_token` in the next request. When `next_page_token` is empty, there are no more results. This pattern is simple, consistent, and well-understood.

**Always include a total count or indicator of remaining results.** Consumers need to know whether more data exists. Include either a `total_count` field (if computing it is cheap) or a boolean `has_more` field. Without this, consumers cannot distinguish "last page" from "exactly page-sized result that happens to end here."

**Support field filtering.** Let consumers request only the fields they need. The Google API Design Guide uses FieldMask; simpler APIs use a `fields` query parameter (`?fields=id,name,email`). This reduces payload size, speeds up responses, and allows the server to skip expensive computations for unrequested fields.

**Provide batch endpoints for bulk operations.** If consumers regularly need to create, update, or delete multiple resources, provide a batch endpoint rather than forcing them to loop and make N individual requests. This reduces latency, simplifies client code, and lets the server optimize the operation. From Geewax: batch endpoints should support partial success — return individual status codes for each item in the batch so consumers know which operations succeeded and which failed.

---

## 6. Idempotency and Safety

HTTP methods have defined safety and idempotency properties. Respecting these properties is not optional — violating them breaks caches, proxies, retry logic, and consumer expectations.

**Safe methods must not modify state.** From RFC 7231: GET, HEAD, and OPTIONS are safe methods. A GET request must never create, modify, or delete a resource. Consumers, caches, and crawlers all assume safe methods have no side effects. Violating this assumption causes data corruption and unpredictable behavior.

**Idempotent methods must produce the same result on repeat calls.** From RFC 7231: PUT and DELETE are idempotent. Sending the same PUT request twice should leave the resource in the same state. Sending DELETE twice — the second call should return 404 or 204, not an error. Design your implementation so that repeated calls converge to the same outcome.

**POST is neither safe nor idempotent.** Creating a resource with POST can produce a different result on each call (a new resource each time). This means POST requests cannot be blindly retried without risking duplicates. Any API that accepts payments, creates orders, or triggers irreversible side effects must account for this.

**Use idempotency keys for non-idempotent operations.** From Geewax: when a client needs to safely retry a POST (for example, creating a payment), the client sends a unique idempotency key in a request header. The server stores the key and the result; if the same key is sent again, the server returns the stored result instead of processing the request a second time. This is essential for any operation involving money, inventory, or other irreversible effects.

**Define idempotency key semantics clearly.** Document how long the server retains idempotency keys, what happens if the same key is sent with different request bodies, and what scope the key applies to (per-user, per-endpoint, or global). From Geewax: ambiguous idempotency semantics are worse than no idempotency support at all — consumers will make incorrect assumptions.

**Design for safe retries.** Network failures are inevitable. Every operation in your API should have a clear retry story. Safe and idempotent methods can be retried freely. Non-idempotent operations need idempotency keys or other deduplication mechanisms. Document the retry semantics for each endpoint so consumers know what is safe to retry and under what conditions.

**Handle conditional requests for concurrency control.** From Richardson and Amundsen: use ETags and the `If-Match` header to prevent lost updates. When a client retrieves a resource, the response includes an ETag. When the client updates the resource, it sends `If-Match` with the ETag value. If another client modified the resource in the meantime, the server returns 412 Precondition Failed. This prevents the "last writer wins" problem without requiring pessimistic locking.

---

## 7. Authentication, Rate Limiting, and Quotas

An API exposed to the network will be abused. Design authentication, rate limiting, and quotas into the API from the start — retrofitting them is painful and error-prone.

**Use API keys for identification, OAuth 2.0 for authorization.** From Jin, Sahni, and Shevat: API keys identify which application is making the request and are suitable for tracking and rate limiting. OAuth 2.0 handles authorization — determining what a user or application is allowed to do. Do not conflate the two. API keys are not a substitute for proper authorization.

**Never pass credentials in URLs.** API keys and tokens should be sent in headers, not in query parameters. URLs are logged by proxies, servers, and browsers. Credentials in URLs end up in log files, browser history, and referrer headers — all of which are security liabilities.

**Communicate rate limits through standard headers.** Use `X-RateLimit-Limit` (the maximum number of requests in the window), `X-RateLimit-Remaining` (how many requests the client has left), and `X-RateLimit-Reset` (when the window resets). These headers let clients self-regulate and avoid hitting the limit. Proactive clients will throttle themselves when `X-RateLimit-Remaining` is low rather than waiting to receive a 429.

**Return 429 with Retry-After when the limit is exceeded.** From the Google API Design Guide: when a client exceeds the rate limit, return HTTP 429 Too Many Requests with a `Retry-After` header indicating when the client can retry. Never return a generic 500 for rate limiting — the client cannot distinguish a rate-limit error from a server failure and cannot respond appropriately.

**Enforce per-user or per-API-key quotas.** Quotas prevent a single consumer from monopolizing API resources. Define quotas for different tiers of access (free, standard, premium) and enforce them consistently. Document the quota limits and what happens when they are exceeded.

**Distinguish between rate limits and quotas.** From Jin, Sahni, and Shevat: rate limits control the speed of requests (e.g., 100 requests per minute). Quotas control the total volume of usage (e.g., 10,000 API calls per month). Both are necessary. A consumer can be within their rate limit but exceed their monthly quota, or vice versa. Communicate both clearly and enforce them independently.

**Degrade gracefully under load.** When the API is under pressure, prefer returning partial results or cached data with appropriate headers over returning errors. If degradation is not possible, shed load predictably — enforce rate limits and quotas rather than letting the server crash and returning unhelpful 500 errors to all consumers.

---

## 8. Documentation and Developer Experience

An API without documentation is an API without users. Documentation is not supplementary — it is part of the product.

**Generate documentation from the specification.** From Geewax: the API specification (OpenAPI, protobuf, GraphQL SDL) is the source of truth. Generate documentation from it automatically. Manually maintained documentation drifts from reality and becomes a source of misinformation. When the spec changes, the docs change with it.

**Document every endpoint completely.** Every endpoint needs: a description of what it does, the request format with required and optional fields, the response format with all fields described, the possible error responses and their meanings, and the authentication requirements. From Jin, Sahni, and Shevat: documentation that omits error cases or authentication details forces developers into trial-and-error.

**Provide request and response examples.** Abstract schema descriptions are necessary but not sufficient. Concrete examples show developers what a real request looks like and what they should expect back. Include examples for success cases and for common error cases. From Jin, Sahni, and Shevat: examples are the first thing most developers look at — they orient the reader before the reference material provides precision.

**Use consistent terminology across the entire API surface.** From the Google API Design Guide: if you call it a "user" in one endpoint, do not call it an "account" in another. If you use `created_at` for timestamps in one resource, do not use `creation_date` in another. Inconsistency forces developers to read documentation for every endpoint instead of relying on learned patterns.

**Maintain a changelog.** Document every change to the API, especially breaking changes. Each changelog entry should state what changed, when it changed, and what consumers need to do (if anything). From Geewax: the changelog is the primary tool for communicating API evolution to consumers.

**Make onboarding fast.** A new developer should be able to make their first successful API call within minutes. Provide a quickstart guide, working examples, and a sandbox or test environment. The time from "I found your API" to "I made a successful call" is the most important metric for developer experience. From Jin, Sahni, and Shevat: every hour of friction during onboarding costs you potential adopters.

**Version your documentation alongside your API.** Consumers on v1 need v1 documentation, not v2 documentation that describes fields and behaviors that do not exist in their version. From Geewax: documentation that does not match the version the consumer is using is worse than no documentation — it actively misleads.

---

## Applying This Skill

When designing a new API, start with the contract — write the OpenAPI spec or protobuf schema first and validate it with consumers before implementing. When reviewing an API, check each of these areas systematically:

- Correct HTTP status codes for all success and error paths
- Consistent resource naming and pluralization
- Pagination support on all collection endpoints
- Backward compatibility of any changes to existing endpoints
- Complete documentation with examples and error cases
- Idempotency guarantees for state-changing operations
- Rate limiting headers and quota documentation

When evolving an existing API, treat every change as potentially breaking until you have confirmed otherwise — verify that no existing fields have changed type, meaning, or optionality.

Call out violations by name: "this endpoint returns 200 for validation errors — use 400 with RFC 7807 Problem Details," or "this field was renamed from `user_name` to `username` — that is a breaking change." Specific, grounded feedback is actionable feedback.

Prioritize the consumer's experience over implementation convenience. If a design decision makes the server simpler but the client harder, reconsider. The server is written once; client code is written by every consumer.
