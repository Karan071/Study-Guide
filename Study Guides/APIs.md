# APIs: A Practical Backend Engineer's Study Guide

Audience: Developers who design, build, test, and operate APIs.
Scope: HTTP fundamentals, resource design, contracts, security, reliability, and operations.
Last reviewed: 2026-10-11. Standards and platform behavior change; use the linked specifications when making implementation decisions.

This guide uses simple language for technical ideas. An **API** (Application Programming Interface) is a defined way for software to request data or actions from other software. The examples focus on HTTP APIs, especially REST-style JSON APIs.

## Table of Contents

0. [API System Map](#0-api-system-map)
1. [HTTP Request and Response](#1-http-request-and-response)
2. [REST and Resource Design](#2-rest-and-resource-design)
3. [HTTP Methods and Idempotency](#3-http-methods-and-idempotency)
4. [Status Codes and Error Responses](#4-status-codes-and-error-responses)
5. [Request Validation and Data Formats](#5-request-validation-and-data-formats)
6. [Authentication and Authorization](#6-authentication-and-authorization)
7. [Pagination, Filtering, and Sorting](#7-pagination-filtering-and-sorting)
8. [API Contracts and OpenAPI](#8-api-contracts-and-openapi)
9. [Versioning and Compatibility](#9-versioning-and-compatibility)
10. [API Security](#10-api-security)
11. [Rate Limits and Resource Protection](#11-rate-limits-and-resource-protection)
12. [Caching](#12-caching)
13. [Webhooks](#13-webhooks)
14. [Testing APIs](#14-testing-apis)
15. [Observability and Reliability](#15-observability-and-reliability)
16. [Choosing REST, GraphQL, or gRPC](#16-choosing-rest-graphql-or-grpc)
17. [Production Checklist](#17-production-checklist)
18. [Self-Check Questions](#18-self-check-questions)
19. [Master Resource List](#19-master-resource-list)

## 0. API System Map

An API request crosses several boundaries. Each boundary needs a clear contract and a clear owner.

```text
Client
  |
  | HTTPS request: method + URL + headers + optional body
  v
Gateway / load balancer
  | TLS, request limits, routing, coarse rate limiting
  v
API service
  | authenticate -> authorize -> validate -> execute
  v
Database / other services
  |
  v
API service -> HTTPS response: status + headers + optional body -> Client
```

| Concern | Main question | Typical mechanism |
| --- | --- | --- |
| Contract | What can the client send and expect back? | OpenAPI, schemas, documentation |
| Transport | How are bytes exchanged? | HTTP over TLS |
| Identity | Who is making the request? | Session cookie, API key, or access token |
| Permission | Is this identity allowed to do this action on this object? | Policy and ownership checks |
| Correctness | Is the input valid and is the operation safe to repeat? | Validation, transactions, idempotency |
| Reliability | What happens during overload or dependency failure? | Timeouts, limits, retries, monitoring |

### A basic request lifecycle

```text
1. Parse the request and enforce size / format limits.
2. Authenticate the caller, if the endpoint is protected.
3. Authorize the requested action and object.
4. Validate parameters and the request body.
5. Run the operation, using transactions where needed.
6. Return the documented status, headers, and response body.
7. Record useful logs and metrics without logging secrets.
```

Do not rely on the client to enforce security rules. Clients can be modified or bypassed; the API service must check every protected operation.

## 1. HTTP Request and Response

HTTP is a request-response protocol. A client sends a request; a server returns a response. HTTPS is HTTP protected by TLS, which provides encryption in transit and helps verify the server's identity.

### Core mechanism

An HTTP request contains a method, target URL, headers, and sometimes a body. A response contains a status code, headers, and sometimes a body.

```http
GET /v1/books/42 HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer <access-token>
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: private, no-store

{
  "id": "42",
  "title": "Example"
}
```

- **Method** says what kind of operation is requested.
- **URL** identifies the target resource or operation.
- **Headers** carry metadata such as content type, credentials, or cache rules.
- **Body** carries data, commonly JSON for an API.
- **Status code** gives the broad result. The body can give more detail.

### Engineering blueprint

- Serve production APIs over HTTPS only.
- Use `Content-Type: application/json` when sending JSON.
- Use `Accept` to describe response formats the client can process when content negotiation is supported.
- Document required and optional headers, parameters, and body fields.
- Keep credentials out of URLs. URLs are often recorded in browser history, logs, and analytics.

### Production pitfalls

- Treating a status code as the entire response contract; clients also need stable response fields and headers.
- Sending a body with a `GET` request and expecting every proxy or server to interpret it the same way.
- Assuming the network delivers a request exactly once. Connections can fail after the server acts but before the client receives the response.
- Using HTTP without TLS, exposing credentials and data to network observers.

### SDE best practices

- Make the request and response shapes explicit in the API contract.
- Keep headers and status codes meaningful and consistent across endpoints.
- Set request size, connection, and processing time limits appropriate to the service.
- Treat all client-provided values as untrusted input.

### Resources

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9112: HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112)
- [MDN: HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP)

## 2. REST and Resource Design

REST is an architectural style for networked systems. A REST-style HTTP API usually models data and actions as **resources** identified by URLs, and uses standard HTTP methods to interact with them. Many APIs called “REST” use only some REST constraints; consistency and a clear contract matter more than the label.

### Core mechanism

A resource is a thing the API exposes, such as a book, order, or account. Use nouns in paths and make the URL hierarchy understandable.

```text
GET    /v1/books          list books
POST   /v1/books          create a book
GET    /v1/books/42       read one book
PATCH  /v1/books/42       change selected fields
DELETE /v1/books/42       delete a book
GET    /v1/books/42/loans list loans for a book
```

The path should identify the resource. The method should express the operation. Avoid paths that expose database table names or internal implementation details.

### Engineering blueprint

- Choose a consistent plural or singular naming convention for collections.
- Use stable public identifiers; do not expose sequential database keys if they leak sensitive information or create unwanted coupling.
- Represent relationships clearly, but avoid deeply nested paths that are hard to use.
- Use `POST` for collection creation or operations that do not fit a standard update.
- Use `PATCH` for partial updates and `PUT` when replacing the full representation is intended.
- Keep business rules on the server; the client requests an operation but does not dictate internal processing.

### Actions that do not fit CRUD

Some operations are state transitions, not field edits. Model them as a sub-resource or a custom action rather than letting clients `PATCH` a `status` field directly, so the server can enforce the transition rules:

```text
POST /v1/orders/ord_123/cancel        # custom action (Google AIP-136 style: /orders/ord_123:cancel)
POST /v1/orders/ord_123/refunds       # create a refund resource
```

### Long-running operations

If work takes longer than a normal request timeout (exports, video processing, bulk imports), accept the request and return an operation resource the client can poll:

```http
POST /v1/exports
→ 202 Accepted
Location: /v1/operations/op_789

GET /v1/operations/op_789
→ 200 OK
{"id": "op_789", "status": "running", "progress": 0.4}

GET /v1/operations/op_789
→ 200 OK
{"id": "op_789", "status": "succeeded", "result": {"url": "/v1/exports/exp_55"}}
```

Document the possible states (`pending`, `running`, `succeeded`, `failed`, `cancelled`), how long operation records are kept, and whether a webhook is sent on completion so clients do not have to poll.

### Production pitfalls

- Creating a different URL or response style for every resource.
- Returning internal database objects directly, making schema changes break clients.
- Putting action names in every URL, for example `/getBook` or `/deleteBook`, when HTTP methods already express the operation.
- Building a “REST” API that returns overly large or inconsistent objects without a documented reason.

### SDE best practices

- Design around client tasks and domain concepts, not only around database tables.
- Use the same field names and representations for the same concept across endpoints.
- Keep public API models separate from persistence models when their requirements differ.
- Make resource ownership and access rules explicit.

### Resources

- [Fielding dissertation: Architectural Styles and the Design of Network-based Software Architectures](https://ics.uci.edu/~fielding/pubs/dissertation/top.htm)
- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)
- [Google AIP-121: Resource-oriented design](https://google.aip.dev/121)
- [Google AIP-136: Custom methods](https://google.aip.dev/136)
- [Google AIP-151: Long-running operations](https://google.aip.dev/151)

## 3. HTTP Methods and Idempotency

HTTP methods have defined semantics. Respecting those semantics lets clients, proxies, and tools handle requests more safely.

### Core mechanism

| Method | Common API use | Safe? | Idempotent? |
| --- | --- | --- | --- |
| `GET` | Read a resource | Yes | Yes |
| `HEAD` | Read response headers without the body | Yes | Yes |
| `POST` | Create a resource or run an operation | No | Not by default |
| `PUT` | Create or replace a resource at a known URL | No | Yes |
| `PATCH` | Apply a partial change | No | Not necessarily |
| `DELETE` | Delete a resource | No | Yes |

**Safe** means the client is not asking the server to change the resource state. **Idempotent** means repeating the same request has the same intended effect as making it once. An idempotent request can still return a different response on a later attempt; for example, the first `DELETE` may return `204`, while a repeated one returns `404`.

### Engineering blueprint

For a payment or other high-impact `POST`, accept an idempotency key if clients may retry after a timeout:

```http
POST /v1/payments
Idempotency-Key: 2f5c...
Content-Type: application/json

{"order_id":"ord_123","amount":2500,"currency":"USD"}
```

The server should associate the key with the authenticated caller, operation, and request result for a documented retention period. A repeated key with the same request should return the original outcome rather than performing the action again. A repeated key with a different request should be rejected.

A minimal server-side implementation:

```text
idempotency_keys(
  caller_id, key,               -- unique together
  request_hash,                 -- hash of method + path + body
  status,                       -- in_progress | completed
  response_code, response_body,
  created_at                    -- used for expiry, e.g. 24 hours
)

on request(caller, key, req):
  row = insert (caller, key, hash(req), 'in_progress') on conflict do nothing
  if insert succeeded:
      result = execute(req)            -- same DB transaction as the row update where possible
      update row set status='completed', response=result
      return result
  existing = select row
  if existing.request_hash != hash(req): return 422 (key reused with different request)
  if existing.status == 'in_progress':  return 409 (retry later)
  return existing.response             -- replay the original outcome
```

### Optimistic concurrency with `ETag` and `If-Match`

Two clients can read the same resource and both try to update it; without a check, the second write silently overwrites the first ("lost update"). Use the resource version as a precondition:

```http
GET /v1/books/42
→ 200 OK
ETag: "v7"

PATCH /v1/books/42
If-Match: "v7"
Content-Type: application/merge-patch+json

{"title": "New title"}
```

If the stored version is still `v7`, the server applies the change and returns the new `ETag`. If another client already changed it, the server returns `412 Precondition Failed`, and the client re-reads and retries. In SQL this is often a single conditional update: `UPDATE books SET ..., version = version + 1 WHERE id = 42 AND version = 7`, then checking that one row was affected.

### Production pitfalls

- Automatically retrying non-idempotent operations and creating duplicate payments, orders, or messages.
- Treating an idempotency key as authentication. It is not a secret or a permission.
- Reusing one key for unrelated operations or users.
- Claiming an operation is idempotent when each retry sends another email or triggers another external side effect.

### SDE best practices

- Define retry behavior for each operation, including what happens after timeouts.
- Use idempotency keys for retryable operations that can create harmful duplicates.
- Store the idempotency record and business change atomically where possible.
- Set limits and expiry rules for stored idempotency keys.

### Resources

- [RFC 9110: Safe and Idempotent Methods](https://www.rfc-editor.org/rfc/rfc9110#section-9.2)
- [RFC 5789: PATCH Method](https://www.rfc-editor.org/rfc/rfc5789)
- [RFC 7396: JSON Merge Patch](https://www.rfc-editor.org/rfc/rfc7396)
- [IETF draft: The Idempotency-Key HTTP Header Field](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/)
- [Stripe: Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency)

## 4. Status Codes and Error Responses

Status codes give clients a standard, broad description of what happened. Use the code that matches the result, then provide a stable error body with details safe for the client to see.

### Core mechanism

| Code | Meaning | Typical use |
| --- | --- | --- |
| `200 OK` | Request succeeded | Successful read or update with a response body |
| `201 Created` | Resource created | Successful creation; often include a `Location` header |
| `202 Accepted` | Work accepted, not finished | Background or asynchronous operation |
| `204 No Content` | Request succeeded without a response body | Successful delete or update with no representation |
| `400 Bad Request` | Request is malformed or invalid | Invalid syntax or request-level validation failure |
| `401 Unauthorized` | Authentication is missing or invalid | Missing, expired, or invalid credentials |
| `403 Forbidden` | Caller is not allowed | Authenticated caller lacks permission |
| `404 Not Found` | Resource was not found or is not disclosed | Missing resource |
| `405 Method Not Allowed` | Resource exists but does not support this method | `DELETE` on a read-only resource; include an `Allow` header |
| `409 Conflict` | Request conflicts with current state | Duplicate or incompatible state transition |
| `412 Precondition Failed` | A request precondition failed | Conditional update using `If-Match` |
| `415 Unsupported Media Type` | Server cannot process the request body's format | Client sent XML or form data where JSON is required |
| `422 Unprocessable Content` | Content is understood but cannot be processed | Semantic validation failure, if used consistently |
| `429 Too Many Requests` | Client exceeded a limit | Rate or quota limit |
| `500 Internal Server Error` | Unexpected server failure | Generic server-side failure |
| `502 Bad Gateway` | A gateway or proxy got an invalid response upstream | Upstream service crashed or closed the connection |
| `503 Service Unavailable` | Service temporarily cannot handle request | Overload or planned temporary unavailability |
| `504 Gateway Timeout` | A gateway or proxy did not get a response in time | Upstream service exceeded the gateway timeout |

A quick rule for clients: `4xx` means "change the request before retrying" (except `408`, `429`, and sometimes `409`); `5xx` means "the server failed" and the request may be retried if the operation is idempotent or carries an idempotency key.

`401` and `403` are different: `401` means the request lacks valid authentication; `403` means the identity is known but the action is not allowed. An API may return `404` instead of `403` when revealing that a protected object exists would expose sensitive information.

### Engineering blueprint

Use the Problem Details format (RFC 9457) or a documented equivalent. Keep the shape stable:

```http
HTTP/1.1 422 Unprocessable Content
Content-Type: application/problem+json

{
  "type": "https://api.example.com/problems/invalid-request",
  "title": "Request validation failed",
  "status": 422,
  "detail": "One or more fields are invalid.",
  "instance": "/v1/books",
  "errors": [
    {"field": "title", "code": "required"}
  ]
}
```

The `errors` extension is an API-specific field; document it. Do not send stack traces, SQL errors, secrets, or internal hostnames to clients.

### Production pitfalls

- Returning `200` with an error object for every result; this breaks standard client handling and monitoring.
- Using `500` for invalid client input.
- Returning different error schemas from different endpoints without need.
- Exposing whether a protected user or object exists through overly detailed errors.

### SDE best practices

- Document the status codes each operation may return.
- Use stable machine-readable error codes; clients should not parse human text to decide behavior.
- Make error messages useful but safe.
- Include a request or trace identifier that support teams can use to find server-side logs.

### Resources

- [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457)
- [RFC 9110: Status Codes](https://www.rfc-editor.org/rfc/rfc9110#section-15)

## 5. Request Validation and Data Formats

Validation checks that incoming data has the expected shape and meaning. The API must validate input even when the user interface already does so.

### Core mechanism

Validation has multiple layers:

1. **Parsing:** Is the body valid JSON or another supported format?
2. **Shape:** Are required fields present and are values the expected types?
3. **Constraints:** Are lengths, ranges, formats, and allowed values valid?
4. **Business rules:** Is the requested change allowed in the current domain state?

Example request schema:

```json
{
  "type": "object",
  "required": ["title"],
  "properties": {
    "title": {"type": "string", "minLength": 1, "maxLength": 200},
    "published": {"type": "boolean"}
  },
  "additionalProperties": false
}
```

Choose whether unknown fields are rejected or ignored and apply the rule consistently. Rejecting unknown fields can catch client mistakes; allowing them can make some additive changes easier. The choice is part of the contract.

### Engineering blueprint

- Set maximum body sizes before parsing.
- Validate path parameters, query parameters, headers, and body fields.
- Normalize data only where the normalization rule is clear and documented.
- Use database constraints as a second line of defense for invariants such as uniqueness.
- Return field-level errors without echoing unsafe input.

### Production pitfalls

- Trusting values because they came from an internal service or a browser form.
- Using loose type coercion that silently changes `false` to a string or truncates numbers.
- Accepting very large strings, arrays, or nested JSON that consume excessive CPU or memory.
- Treating a syntactically valid value as a valid business operation.

### SDE best practices

- Keep validation rules close to the API boundary and share schemas where it is safe to do so.
- Distinguish missing, `null`, empty, and default values; they can have different meanings.
- Use allow-lists for fields a caller may set, especially on update endpoints.
- Validate both before business logic and at the persistence boundary for important invariants.

### Resources

- [JSON Schema](https://json-schema.org/)
- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- [OWASP Input Validation Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html)

## 6. Authentication and Authorization

Authentication (**authn**) establishes who the caller is. Authorization (**authz**) decides what that caller may do. An API often needs both.

### Core mechanism

Common credential types include:

- **Session cookie:** A browser sends a cookie that identifies a server-side session.
- **Bearer access token:** A client sends a token in `Authorization: Bearer ...`.
- **API key:** A service credential that identifies a project or client; it does not automatically identify an end user.

After verifying the credential, the API checks permissions for the requested action and resource:

```text
principal = verify_credential(request)
resource = load_resource(request.path.id)
if not allowed(principal, "book:update", resource):
    return 403
```

### Engineering blueprint

- Use an established identity provider and standard protocols where possible.
- Send credentials in headers or secure cookies, never query strings.
- Require authorization checks on every protected operation, including reads.
- Check ownership and tenant boundaries using the authenticated principal, not a caller-supplied owner ID alone.
- Use short-lived, scoped credentials and provide rotation or revocation for long-lived credentials.

### Production pitfalls

- Assuming that successful authentication means the caller can access every object.
- Trusting a `user_id`, `role`, or `tenant_id` supplied in an untrusted request body.
- Using API keys as user passwords or granting them broader access than required.
- Logging access tokens, session cookies, or API keys.

### SDE best practices

- Deny access by default and grant only the permissions required.
- Test object-level authorization, not only whether a request is logged in.
- Return `401` for invalid or absent credentials and `403` when an authenticated caller is denied, unless the API intentionally hides resource existence.
- Keep authorization policy consistent across API routes and background operations.

For a deeper guide to OAuth, OIDC, sessions, and tokens, see [AuthenticationConcepts.md](./AuthenticationConcepts.md).

### Resources

- [OWASP API Security Top 10](https://owasp.org/API-Security/)
- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- [AuthenticationConcepts.md](./AuthenticationConcepts.md)

## 7. Pagination, Filtering, and Sorting

Collections can grow large. Pagination lets a client retrieve a manageable part of a collection instead of asking for every item at once.

### Core mechanism

Two common pagination styles are:

- **Offset pagination:** `?limit=20&offset=40`. Simple and useful for small or relatively stable datasets, but large offsets can be slow and inserts can shift results between pages.
- **Cursor pagination:** `?limit=20&cursor=...`. The server returns an opaque cursor for continuing from a specific position. It is usually more stable for large or changing collections.

Example response:

```json
{
  "items": [{"id": "bk_1"}, {"id": "bk_2"}],
  "next_cursor": "opaque-value",
  "has_more": true
}
```

Cursor pagination is usually implemented as **keyset pagination**. The cursor encodes the sort key of the last item returned, and the next query starts after it:

```sql
-- Offset: the database still reads and discards the first 10,000 rows.
SELECT * FROM books ORDER BY created_at DESC, id DESC LIMIT 20 OFFSET 10000;

-- Keyset: an index on (created_at, id) lets the database seek directly to the position.
SELECT * FROM books
WHERE (created_at, id) < (:last_created_at, :last_id)
ORDER BY created_at DESC, id DESC
LIMIT 21;   -- fetch one extra row to compute has_more
```

Encode `(last_created_at, last_id)` into the cursor (for example, base64 JSON, optionally signed) so clients treat it as opaque. Row-value comparison is supported by PostgreSQL and MySQL; other databases may need the expanded form `created_at < ? OR (created_at = ? AND id < ?)`.

### Engineering blueprint

- Set a default page size and a maximum page size.
- Define deterministic ordering, including a unique tie-breaker such as an ID.
- Whitelist sortable and filterable fields; do not insert arbitrary client strings into database queries.
- Document whether totals are exact, approximate, or omitted.
- Make cursors opaque and validate them. A cursor is not authorization.

### Production pitfalls

- Returning unbounded collections.
- Using unstable ordering, causing duplicate or missing items between pages.
- Allowing arbitrary filters that create expensive queries or expose fields.
- Treating a cursor as proof that the caller can access the underlying records.

### SDE best practices

- Prefer cursor pagination for large, frequently changing collections.
- Apply authorization and tenant filtering before pagination.
- Design indexes to support common filter and sort patterns.
- Explain how clients know whether another page exists.

### Resources

- [Google AIP-158: Pagination](https://google.aip.dev/158)
- [JSON:API pagination](https://jsonapi.org/format/#fetching-pagination)
- [Use The Index, Luke: Paging through results without OFFSET](https://use-the-index-luke.com/no-offset)

## 8. API Contracts and OpenAPI

An API contract describes how clients use an API: operations, inputs, outputs, security, and errors. A clear contract reduces guesswork and helps independent teams work safely.

### Core mechanism

OpenAPI is a language-independent format for describing HTTP APIs. Tools can use an OpenAPI document to generate documentation, clients, test scaffolding, and validation.

```yaml
paths:
  /v1/books/{bookId}:
    get:
      operationId: getBook
      parameters:
        - name: bookId
          in: path
          required: true
          schema:
            type: string
      responses:
        "200":
          description: Book found
        "404":
          description: Book not found
```

This abbreviated example omits reusable schemas and security configuration; production specifications should describe the real request and response bodies too.

### Engineering blueprint

- Decide whether the contract is design-first (write the specification, then implement) or code-first (generate the specification from code).
- Keep the published contract aligned with deployed behavior.
- Describe authentication, field constraints, error responses, pagination, and examples.
- Use generated checks or compatibility tests to detect differences between implementation and specification.

### Production pitfalls

- Treating documentation as optional or allowing it to drift.
- Publishing a specification that lists response codes but leaves response schemas vague.
- Generating a client from an incomplete or inaccurate contract.
- Exposing internal-only endpoints or fields in a public API document.

### SDE best practices

- Review API changes as contract changes, not only code changes.
- Use stable `operationId` values and reusable schemas.
- Add realistic examples that do not contain secrets or production data.
- Define how the specification is versioned and published.

### Resources

- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- [Swagger tools](https://swagger.io/tools/)

## 9. Versioning and Compatibility

An API change is **backward-compatible** when existing clients continue to work without changes. Compatibility matters because clients may not upgrade at the same time as the server.

### Core mechanism

Usually safe additions include an optional request field or a response field that clients are expected to ignore when unknown. Often breaking changes include removing or renaming fields, changing a field's type or meaning, or making an optional input required.

Common versioning approaches include:

- Version in the URL, such as `/v1/books`.
- Version in a media type or header.
- Evolving one version with compatible changes and retiring old behavior through a deprecation process.

No approach removes the need to communicate changes. Choose a strategy and apply it consistently.

Signal deprecation in responses so clients can detect it automatically:

```http
HTTP/1.1 200 OK
Deprecation: @1767225600
Sunset: Wed, 30 Jun 2027 23:59:59 GMT
Link: <https://api.example.com/docs/migrate-v2>; rel="deprecation"
```

`Deprecation` (RFC 9745) says when the resource was or will be deprecated, as a Unix timestamp. `Sunset` (RFC 8594) says when it may stop responding.

### Engineering blueprint

- Define compatibility rules before the API has many clients.
- Prefer additive changes when possible.
- Announce deprecations, provide migration guidance, and set a realistic retirement date.
- Track usage of older versions to understand who will be affected.
- Keep old versions isolated enough that they can be retired safely.

### Production pitfalls

- Renaming a response field because the new name seems clearer.
- Changing a field from seconds to milliseconds, or changing units without changing its name.
- Adding a new enum value when clients may fail on unknown values.
- Assuming all clients upgrade immediately after a server deployment.

### SDE best practices

- Treat response fields as potentially unknown to clients; document whether clients should ignore them.
- Use explicit units and stable meanings in field names or descriptions.
- Run contract or schema compatibility checks in CI.
- Make deprecation visible in documentation and response headers where appropriate.

### Resources

- [Google AIP-180: Backwards compatibility](https://google.aip.dev/180)
- [RFC 8594: The Sunset HTTP Header](https://www.rfc-editor.org/rfc/rfc8594)
- [RFC 9745: The Deprecation HTTP Response Header Field](https://www.rfc-editor.org/rfc/rfc9745)

## 10. API Security

API security protects data and operations from misuse. A valid token is not enough: endpoints must also control access to individual objects, fields, and actions.

### Core mechanism

Use defense in depth:

- **Transport:** Require TLS.
- **Authentication:** Verify credentials consistently.
- **Authorization:** Check action, object, and tenant access.
- **Input handling:** Validate and constrain all untrusted input.
- **Output handling:** Return only fields the caller is allowed to see.
- **Operations:** Limit resource use, rotate secrets, and monitor suspicious behavior.

### Engineering blueprint

- Follow the OWASP API Security Top 10 as a threat-model checklist.
- Prevent broken object-level authorization by checking permission on each referenced object.
- Prevent broken object property-level authorization by allow-listing writable and readable fields.
- Use parameterized database queries; never build SQL by concatenating untrusted input.
- Restrict cross-origin browser access with an explicit CORS policy. CORS is not authentication.
- Protect cookie-authenticated state-changing requests against CSRF.
- Keep secrets in a secret manager or protected runtime configuration, not source code.

### Production pitfalls

- **BOLA / IDOR:** Changing an ID in the URL reveals another user's record.
- **Mass assignment:** A caller sets an internal field such as `is_admin` through a generic update endpoint.
- **Excessive data exposure:** An endpoint returns private fields that the client interface happens not to display.
- **SSRF:** A feature fetches a user-provided URL and can reach internal services or cloud metadata endpoints.
- **Injection:** User input changes the meaning of a database query or command.
- Treating CORS as a server-side permission check.

### SDE best practices

- Test access using multiple users, tenants, and permission levels.
- Allow-list fields accepted by create and update operations.
- Keep dependencies and API gateways updated; review security advisories.
- Rate-limit sensitive operations such as login, password reset, and expensive searches.
- Log security events safely, but never log credentials or full sensitive payloads.

### Resources

- [OWASP API Security Top 10](https://owasp.org/API-Security/)
- [OWASP API Security Top 10 (2023 edition)](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)

## 11. Rate Limits and Resource Protection

Rate limiting controls how frequently a client can call an API. It helps protect availability, control costs, and limit abuse. A rate limit is not a replacement for authorization or capacity planning.

### Core mechanism

A limit may be applied per account, API key, user, IP address, tenant, or endpoint. Use the identity that best matches the risk; IP-only limits can affect many legitimate users behind the same network.

When a client exceeds a limit, return `429 Too Many Requests`. If useful, include `Retry-After` to tell the client when to try again. Some APIs also return documented limit and reset headers; a standard `RateLimit` / `RateLimit-Policy` header format is being developed by the IETF but is still a draft.

Common algorithms:

| Algorithm | How it works | Trade-off |
| --- | --- | --- |
| Fixed window | Count requests per clock window (for example, per minute) | Simple; allows a burst of 2x at window boundaries |
| Sliding window | Weight the previous window's count by how much of it overlaps | Smoother; slightly more state |
| Token bucket | Tokens refill at a steady rate up to a capacity; each request spends one | Allows controlled bursts; the most common choice |
| Leaky bucket | Requests drain from a queue at a fixed rate | Smooth output rate; adds queueing delay |

Token bucket in pseudocode:

```text
capacity = 100            # maximum burst
refill_rate = 10          # tokens per second

on request(client):
  b = buckets[client]
  now = clock()
  b.tokens = min(capacity, b.tokens + (now - b.last) * refill_rate)
  b.last = now
  if b.tokens >= 1:
      b.tokens -= 1
      allow
  else:
      reject 429, Retry-After = ceil((1 - b.tokens) / refill_rate)
```

In a multi-instance service, keep the bucket in a shared store (for example, Redis with an atomic Lua script) or enforce the limit at the gateway.

Client retry delay with exponential backoff and "full jitter":

```text
delay = random(0, min(max_delay, base_delay * 2^attempt))
```

The randomness stops many clients that failed at the same moment from retrying at the same moment.

### Engineering blueprint

- Set different limits for different operations based on cost and abuse risk.
- Bound concurrent requests, payload size, execution time, and expensive query complexity.
- Use shared or gateway-level counters when multiple service instances must enforce one limit.
- Decide what happens when a limiter dependency fails; the choice should reflect the risk of allowing traffic through.
- Return a clear error that tells clients whether and when retrying is appropriate.

### Production pitfalls

- Limiting only by IP and blocking whole workplaces or mobile networks.
- Applying a limit after expensive work has already happened.
- Returning `429` without explaining retry behavior.
- Retrying immediately in a tight loop and increasing overload.

### SDE best practices

- Use exponential backoff with jitter for retryable responses.
- Set a maximum retry count and an overall time budget.
- Make limits observable and adjustable without an emergency code deployment where possible.
- Consider per-user fairness and tenant-level quotas in addition to global protection.

### Resources

- [RFC 6585: 429 Too Many Requests](https://www.rfc-editor.org/rfc/rfc6585#section-4)
- [IETF draft: RateLimit header fields for HTTP](https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/)
- [AWS Builders' Library: Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/)

## 12. Caching

Caching stores a response for reuse, reducing latency and server load. HTTP defines headers that let clients and intermediaries decide whether a response can be reused.

### Core mechanism

Important response headers include:

- `Cache-Control: max-age=60` allows reuse while the response is fresh for 60 seconds.
- `Cache-Control: no-store` says not to store the response.
- `ETag` gives a representation validator. A client can send `If-None-Match` to ask whether it changed.
- `Vary` says which request headers affect the selected representation.

Example conditional request:

```http
GET /v1/books/42
If-None-Match: "book-version-7"
```

If unchanged, the server can return `304 Not Modified` without sending the body again.

### Engineering blueprint

- Define cache policy for each response, especially for personalized or sensitive data.
- Use validators such as `ETag` when clients benefit from checking for changes.
- Ensure cache keys distinguish requests that produce different representations.
- Invalidate or version cached data when the underlying resource changes.

### Production pitfalls

- Caching private data in a shared cache.
- Setting `Cache-Control: public` for responses that depend on a user's credentials.
- Forgetting `Vary` when content changes based on headers such as `Accept-Encoding`.
- Assuming that a CDN or browser will cache a response without explicit, correct policy.

### SDE best practices

- Use `no-store` for sensitive responses when storage is not appropriate.
- Test caching behavior with authenticated and unauthenticated requests.
- Set freshness durations based on how stale the data may safely be.
- Measure cache hit rate and stale-data impact; caching adds consistency trade-offs.

### Resources

- [RFC 9111: HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111)
- [MDN: HTTP Caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)

## 13. Webhooks

A webhook is an HTTP request sent by one system to notify another system that an event happened. Unlike polling, the receiver does not need to repeatedly ask for updates.

### Core mechanism

```text
Provider records event
  -> sends signed HTTPS POST to registered receiver
  -> receiver verifies signature and stores event
  -> receiver quickly returns a success response
  -> receiver processes event asynchronously
```

Webhook delivery is commonly **at least once**: a provider retries when it does not receive a successful response. The same event may therefore arrive more than once.

### Engineering blueprint

- Sign the exact request body with a shared secret or another documented verification method.
- Include an event ID, event type, and event creation time.
- Verify the signature using a constant-time comparison and support secret rotation.
- Store or enqueue the event durably before returning success.
- Make event handling idempotent using the event ID.
- Document retry schedules, timeout expectations, event ordering, and how to replay events.

Receiver-side signature check (Python sketch; header names vary by provider):

```python
import hmac, hashlib, time

def verify(raw_body: bytes, timestamp: str, signature: str, secret: bytes) -> bool:
    # Reject old deliveries to limit replay attacks.
    if abs(time.time() - int(timestamp)) > 300:
        return False
    signed = timestamp.encode() + b"." + raw_body          # sign the raw bytes, not re-serialized JSON
    expected = hmac.new(secret, signed, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature)        # constant-time comparison
```

After verification, insert the event ID into a table with a unique constraint. If the insert fails because the ID already exists, the event is a duplicate: return `2xx` and skip processing.

### Production pitfalls

- Trusting a webhook because it came from an expected IP address alone.
- Processing slowly in the request handler and timing out before acknowledging.
- Assuming events arrive once or in order.
- Parsing and reserializing JSON before signature verification when the signature is over the original raw bytes.
- Allowing webhook URLs to trigger requests to internal network addresses.

### SDE best practices

- Keep the endpoint public over HTTPS but verify every request cryptographically.
- Store enough delivery state to diagnose retries and support safe replay.
- Use an allow-list and network protections for outbound webhook destinations to reduce SSRF risk.
- Avoid including secrets or unnecessary personal data in event payloads.

### Resources

- [GitHub: Validating webhook deliveries](https://docs.github.com/en/webhooks/using-webhooks/validating-webhook-deliveries)
- [Stripe: Webhooks](https://docs.stripe.com/webhooks)

## 14. Testing APIs

API tests check both the operation's behavior and the public contract. A successful unit test does not prove that deployed routes, authentication, serialization, and dependencies work together.

### Core mechanism

Useful test levels include:

- **Unit tests:** Check isolated validation and business rules.
- **Integration tests:** Check database, queue, or service interactions.
- **Contract tests:** Check that implementation and client expectations match the API contract.
- **End-to-end tests:** Send real HTTP requests through the deployed or test application stack.
- **Load and resilience tests:** Measure behavior under expected traffic and dependency failures.

### Engineering blueprint

For each endpoint, test:

1. A normal successful request.
2. Missing, malformed, boundary, and unknown input.
3. Missing, invalid, and insufficient credentials.
4. Access to another user's or tenant's objects.
5. Every documented error status and response shape.
6. Retry, duplicate, timeout, and concurrency behavior where relevant.

### Production pitfalls

- Testing only the happy path.
- Mocking the authorization layer in every test and never testing real access checks.
- Depending on shared production data or secrets in automated tests.
- Writing brittle tests that assert incidental header order or exact human-readable error wording.

### SDE best practices

- Use isolated test data and deterministic clocks or IDs when practical.
- Add regression tests for security and data-integrity bugs.
- Validate the OpenAPI document and generate contract checks in CI.
- Use production-like deployments for end-to-end tests, without copying sensitive production data.

### Resources

- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- [Pact: Contract testing](https://docs.pact.io/)
- [OWASP Web Security Testing Guide](https://owasp.org/www-project-web-security-testing-guide/)

## 15. Observability and Reliability

Observability is the ability to understand a service's behavior from its outputs. API operations need logs, metrics, and traces that help teams find failures without exposing sensitive data.

### Core mechanism

- **Logs** record individual events and useful context.
- **Metrics** summarize behavior over time, such as request rate, error rate, and latency.
- **Traces** show how one request moved through multiple services.

A request or trace ID helps connect a client's reported failure to the related server-side records.

### Engineering blueprint

- Measure latency percentiles, request volume, error rate, and saturation (resource pressure).
- Track dependency failures and timeouts separately from API-level failures.
- Propagate trace context across service calls.
- Set timeouts on outbound calls and budgets across the full request path.
- Use bounded retries with backoff and jitter only for operations safe to retry.
- Return a request ID in responses and include it in structured logs.

### Production pitfalls

- Logging access tokens, passwords, personal data, or full request bodies.
- Recording only average latency, hiding slow requests that affect a significant number of clients.
- Retrying every failure and causing a cascading overload.
- Using unbounded user-provided values as metric labels, creating high-cardinality metrics.
- Returning success before durable work has actually been accepted.

### SDE best practices

- Define service-level indicators such as availability and latency for important endpoints.
- Alert on user-visible symptoms, not every internal warning.
- Separate expected client errors from server failures in dashboards.
- Provide graceful overload responses and document client retry behavior.
- Practice incident response and verify that logs and traces contain useful, safe context.

### Resources

- [OpenTelemetry](https://opentelemetry.io/docs/)
- [Google SRE Book: Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [RFC 9110: Retry-After](https://www.rfc-editor.org/rfc/rfc9110#section-10.2.3)

## 16. Choosing REST, GraphQL, or gRPC

These are different API styles and protocols. No one choice is best for every system.

| Style | Good fit | Main trade-off |
| --- | --- | --- |
| REST-style HTTP + JSON | Public APIs, browser and mobile clients, resource-oriented operations | Clients may need several requests or receive fields they do not need |
| GraphQL | Clients need flexible selection of related data through one schema | Server must control query complexity, authorization, and caching carefully |
| gRPC | Internal service-to-service calls, typed contracts, streaming, efficient binary transport | Browser use and direct human inspection can require extra tooling or gateways |

### Core mechanism

- **REST-style APIs** use HTTP resources, methods, and status codes.
- **GraphQL** lets a client request specific fields through a typed query language and schema.
- **gRPC** uses Protocol Buffers and generated service definitions, usually over HTTP/2.

These styles can coexist. For example, a company may expose REST to partners and use gRPC between internal services.

### Engineering blueprint

- Choose based on client needs, team experience, deployment constraints, and operational support.
- Apply authentication, object-level authorization, validation, rate limits, and monitoring regardless of style.
- For GraphQL, cap depth, complexity, and batch size; authorize fields and objects.
- For gRPC, define deadlines, status handling, compatibility rules, and authentication.
- Avoid adding a new protocol unless it solves a real client or operational problem.

### Production pitfalls

- Assuming GraphQL automatically reduces server work; one query can still trigger expensive backend operations.
- Assuming gRPC makes a service reliable without deadlines, retries, and observability.
- Exposing two API styles that drift in permissions or business rules.
- Choosing based only on popularity rather than client and operational requirements.

### Resources

- [GraphQL documentation](https://graphql.org/learn/)
- [gRPC documentation](https://grpc.io/docs/)
- [Protocol Buffers](https://protobuf.dev/)

## 17. Production Checklist

### Contract and design

- [ ] Endpoints, fields, status codes, and error shapes are documented.
- [ ] Resource names and HTTP methods follow a consistent convention.
- [ ] Pagination and maximum response sizes are defined.
- [ ] Compatibility and deprecation rules are documented.

### Security

- [ ] Production traffic uses HTTPS.
- [ ] Credentials are not stored in URLs or logs.
- [ ] Each operation checks authentication and object-level authorization as required.
- [ ] Tenant and ownership checks use trusted identity context.
- [ ] Request sizes, field values, query costs, and outbound destinations are constrained.
- [ ] CORS, CSRF, and cookie settings match the client architecture.

### Reliability

- [ ] Timeouts exist for inbound work and outbound dependencies.
- [ ] Retries are bounded and used only where safe.
- [ ] Retryable create or payment operations have idempotency behavior.
- [ ] Rate limits and overload responses are defined.
- [ ] Important data changes have transaction and consistency rules.

### Testing and operations

- [ ] Positive, validation, authorization, and failure paths have automated tests.
- [ ] Contract checks run as part of CI or release validation.
- [ ] Logs, metrics, and traces support debugging without exposing sensitive data.
- [ ] Alerts cover user-visible service objectives.
- [ ] API changes have a safe rollout and rollback plan.

## 18. Self-Check Questions

Try to answer each question before opening the answer.

<details>
<summary>1. What is the difference between a safe method and an idempotent method? Give an example that is idempotent but not safe.</summary>

A safe method does not ask the server to change state (`GET`, `HEAD`). An idempotent method has the same intended effect whether it runs once or many times. `PUT` and `DELETE` are idempotent but not safe: they change state, but repeating them does not change it further.
</details>

<details>
<summary>2. A client's payment request times out. Why is retrying dangerous, and how do you make it safe?</summary>

The server may have charged the card before the connection failed, so a blind retry can charge twice. The client sends an `Idempotency-Key`; the server stores the key with the result and returns the stored result on a retry instead of charging again.
</details>

<details>
<summary>3. When do you return 401, 403, and 404 for a protected resource?</summary>

`401` when credentials are missing or invalid. `403` when the caller is authenticated but not allowed. `404` instead of `403` when revealing that the resource exists would leak information, for example another tenant's order ID.
</details>

<details>
<summary>4. Why does offset pagination get slow and inconsistent on large tables, and what replaces it?</summary>

The database must read and discard every skipped row, so cost grows with the offset. Inserts or deletes between page requests shift rows, causing duplicates or gaps. Keyset (cursor) pagination filters on the last-seen sort key with an index, so each page costs about the same and is stable under inserts.
</details>

<details>
<summary>5. What is BOLA/IDOR and how do you prevent it?</summary>

Broken Object Level Authorization: the API checks that the caller is logged in but not that the caller may access the specific object ID in the request. Prevent it by loading the object and checking ownership or permissions against the authenticated principal on every request, and by testing with two different users.
</details>

<details>
<summary>6. Two users edit the same record at the same time. How do you avoid the lost-update problem?</summary>

Optimistic concurrency: return a version in an `ETag`, require `If-Match` on updates, and apply the update only if the stored version matches. Otherwise return `412 Precondition Failed` so the client can re-read and retry.
</details>

<details>
<summary>7. Which of these changes are breaking: adding a response field, adding a required request field, renaming a field, adding an enum value?</summary>

Adding a required request field and renaming a field are breaking. Adding a response field is usually safe if clients ignore unknown fields. Adding an enum value can break clients that use exhaustive matching, so document that clients must handle unknown values.
</details>

<details>
<summary>8. Why is webhook delivery "at least once", and what must the receiver do about it?</summary>

The provider cannot tell whether a timed-out delivery was processed, so it retries. The receiver must deduplicate by event ID, acknowledge quickly after durably storing the event, and not assume ordering.
</details>

<details>
<summary>9. Why add jitter to exponential backoff?</summary>

Without jitter, clients that failed together retry together, producing synchronized load spikes that can keep the server overloaded. Random delays spread retries out.
</details>

<details>
<summary>10. When would you pick gRPC over REST, and what do you lose?</summary>

For internal service-to-service calls that need typed contracts, streaming, and efficient binary encoding. You lose easy browser access and human-readable payloads, so public or browser-facing APIs usually stay REST/JSON or go through a gateway.
</details>

## 19. Master Resource List

### HTTP and API design

- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 9457: Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457)
- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)
- [Google API Improvement Proposals](https://google.aip.dev/)

### Security

- [OWASP API Security Top 10](https://owasp.org/API-Security/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [AuthenticationConcepts.md](./AuthenticationConcepts.md)

### Reliability and tooling

- [OpenTelemetry documentation](https://opentelemetry.io/docs/)
- [Pact contract testing](https://docs.pact.io/)
- [MDN Web HTTP reference](https://developer.mozilla.org/en-US/docs/Web/HTTP)
