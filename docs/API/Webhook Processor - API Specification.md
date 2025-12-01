# Webhook Processor - API Specification

## Service-to-Service & External API Documentation

> **Document Type**: API Specification / Interface Contract
> **Version**: 1.0
> **Last Updated**: November 17, 2025
> **Purpose**: Define all API endpoints across services

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Protocol Decisions](#2-protocol-decisions)
3. [Django Service (gRPC Internal)](#3-django-service-grpc-internal)
4. [Auth Service (REST Public)](#4-auth-service-rest-public)
5. [Webhook Service (REST Public)](#5-webhook-service-rest-public)
6. [Error Handling Standards](#6-error-handling-standards)
7. [Authentication & Authorization](#7-authentication--authorization)
8. [API Documentation Standards](#8-api-documentation-standards)

---

## 1. Architecture Overview

### 1.1 Service Boundaries

```
┌──────────────────────────────────────────────────┐
│         EXTERNAL (Public Internet)               │
└────────────────────┬─────────────────────────────┘
                     │ HTTPS/REST
         ┌───────────┴──────────┐
         │                      │
    ┌────▼─────┐         ┌──────▼──────┐
    │   Auth   │         │  Webhook    │
    │ Service  │         │  Service    │
    │ (8000)   │         │  (8001)     │
    └────┬─────┘         └──────┬──────┘
         │                      │
         │       gRPC (Internal Network Only)
         │                      │
         └───────────┬──────────┘
                     │
              ┌──────▼──────┐
              │   Django    │
              │   Service   │
              │   (8002)    │
              │  + Admin UI │
              └─────────────┘
                     │
              ┌──────▼──────┐
              │ PostgreSQL  │
              └─────────────┘
```

### 1.2 Protocol Assignment

| Service  | Public Protocol      | Internal Protocol | Port |
| -------- | -------------------- | ----------------- | ---- |
| Auth     | REST                 | gRPC → Django     | 8000 |
| Webhook  | REST                 | gRPC → Django     | 8001 |
| Django   | None (internal only) | gRPC              | 8002 |
| Admin UI | HTTP (localhost)     | N/A               | 8003 |

---

## 2. Protocol Decisions

### 2.1 Why gRPC for Django?

**Benefits**:

- ✅ **Type Safety**: Protocol Buffers enforce schemas
- ✅ **Performance**: Binary serialization (faster than JSON)
- ✅ **Streaming**: Supports bidirectional streaming (future use)
- ✅ **Code Generation**: Auto-generate client libraries

**Trade-offs**:

- ❌ Not human-readable (use REST for debugging)
- ❌ More complex setup than REST

### 2.2 Why REST for Public APIs?

**Benefits**:

- ✅ **Universal**: Works in browsers, curl, Postman
- ✅ **Human-readable**: JSON responses easy to debug
- ✅ **OpenAPI/Swagger**: Auto-documentation

### 2.3 Decision Matrix

| Use Case                      | Protocol | Reasoning                   |
| ----------------------------- | -------- | --------------------------- |
| Customer login                | REST     | Browser compatibility       |
| FastAPI → Django (get secret) | gRPC     | High frequency, internal    |
| Webhook receiver              | REST     | Third-party compatibility   |
| Django → FastAPI              | N/A      | Django doesn't call FastAPI |
| Admin actions                 | REST     | Low frequency, debugging    |

---

## 3. Django Service (gRPC Internal)

### 3.1 Organizations

#### GetOrganization

```protobuf
rpc GetOrganization(GetOrganizationRequest) returns (Organization) {}

message GetOrganizationRequest {
  string slug = 1;  // e.g., "acme-corp"
}

message Organization {
  string id = 1;  // UUID
  string name = 2;
  string slug = 3;
  string plan = 4;  // "free", "starter", "premium"
  int32 webhook_limit = 5;
  bool is_active = 6;
  string created_at = 7;  // ISO8601
}
```

**Usage**:

```python
# FastAPI calls this to validate org exists
org = django_client.GetOrganization(slug="acme-corp")
```

#### ListOrganizations

```protobuf
rpc ListOrganizations(ListOrganizationsRequest) returns (OrganizationList) {}

message ListOrganizationsRequest {
  int32 page = 1;
  int32 page_size = 2;
  bool is_active = 3;  // Optional filter
}

message OrganizationList {
  repeated Organization organizations = 1;
  int32 total_count = 2;
}
```

#### CreateOrganization

```protobuf
rpc CreateOrganization(CreateOrganizationRequest) returns (Organization) {}

message CreateOrganizationRequest {
  string name = 1;
  string slug = 2;
  string plan = 3;
  int32 webhook_limit = 4;
}
```

#### UpdateOrganization

```protobuf
rpc UpdateOrganization(UpdateOrganizationRequest) returns (Organization) {}

message UpdateOrganizationRequest {
  string id = 1;  // UUID
  optional string plan = 2;
  optional int32 webhook_limit = 3;
  optional bool is_active = 4;
}
```

---

### 3.2 Users

#### GetUser

```protobuf
rpc GetUser(GetUserRequest) returns (User) {}

message GetUserRequest {
  string id = 1;  // UUID
}

message User {
  string id = 1;
  string email = 2;
  string organization_id = 3;
  string role = 4;  // "admin", "member"
  bool is_active = 5;
  string last_login_at = 6;  // ISO8601, nullable
  string created_at = 7;
}
```

#### GetUserByEmail

```protobuf
rpc GetUserByEmail(GetUserByEmailRequest) returns (User) {}

message GetUserByEmailRequest {
  string email = 1;
}
```

**Usage**:

```python
# Auth service: Validate login attempt
user = django_client.GetUserByEmail(email="john@acme.com")
if not user.is_active:
    raise AuthError("Account disabled")
```

#### ListUsers

```protobuf
rpc ListUsers(ListUsersRequest) returns (UserList) {}

message ListUsersRequest {
  string organization_id = 1;  // Filter by org
  optional string role = 2;  // Filter by role
  int32 page = 3;
  int32 page_size = 4;
}

message UserList {
  repeated User users = 1;
  int32 total_count = 2;
}
```

#### UpdateUser

```protobuf
rpc UpdateUser(UpdateUserRequest) returns (User) {}

message UpdateUserRequest {
  string id = 1;
  optional string role = 2;
  optional bool is_active = 3;
}
```

#### UpdateLastLogin

```protobuf
rpc UpdateLastLogin(UpdateLastLoginRequest) returns (UpdateLastLoginResponse) {}

message UpdateLastLoginRequest {
  string user_id = 1;
}

message UpdateLastLoginResponse {
  bool success = 1;
  string last_login_at = 2;  // Updated timestamp
}
```

**Usage**:

```python
# Auth service: After successful login
django_client.UpdateLastLogin(user_id=user.id)
```

---

### 3.3 Webhook Providers

#### GetProviderSecret

```protobuf
rpc GetProviderSecret(GetProviderSecretRequest) returns (ProviderSecret) {}

message GetProviderSecretRequest {
  string organization_slug = 1;
  string provider_name = 2;  // "stripe", "github", etc.
}

message ProviderSecret {
  string id = 1;  // Provider UUID
  string secret = 2;  // Decrypted secret (NEVER log this!)
  bool is_active = 3;
}
```

**Usage**:

```python
# Webhook service: Validate signature
secret = django_client.GetProviderSecret(
    organization_slug="acme-corp",
    provider_name="stripe"
)
is_valid = verify_hmac(payload, signature, secret.secret)
```

**SECURITY NOTE**: This is the ONLY endpoint that returns decrypted secrets!

#### GetProvider

```protobuf
rpc GetProvider(GetProviderRequest) returns (Provider) {}

message GetProviderRequest {
  string id = 1;  // UUID
}

message Provider {
  string id = 1;
  string organization_id = 2;
  string provider_name = 3;
  bool is_active = 4;
  bool has_secret = 5;  // Don't return actual secret!
  string created_at = 6;
}
```

#### ListProviders

```protobuf
rpc ListProviders(ListProvidersRequest) returns (ProviderList) {}

message ListProvidersRequest {
  string organization_id = 1;
  optional bool is_active = 2;
}

message ProviderList {
  repeated Provider providers = 1;
}
```

#### CreateProvider

```protobuf
rpc CreateProvider(CreateProviderRequest) returns (Provider) {}

message CreateProviderRequest {
  string organization_id = 1;
  string provider_name = 2;
  string secret_key = 3;  // Will be encrypted before storage
}
```

#### UpdateProvider

```protobuf
rpc UpdateProvider(UpdateProviderRequest) returns (Provider) {}

message UpdateProviderRequest {
  string id = 1;
  optional string secret_key = 2;  // Update secret (re-encrypted)
  optional bool is_active = 3;
}
```

#### DeleteProvider

```protobuf
rpc DeleteProvider(DeleteProviderRequest) returns (DeleteProviderResponse) {}

message DeleteProviderRequest {
  string id = 1;
}

message DeleteProviderResponse {
  bool success = 1;
}
```

---

### 3.4 Webhook Logs

#### CreateWebhookLog

```protobuf
rpc CreateWebhookLog(CreateWebhookLogRequest) returns (WebhookLog) {}

message CreateWebhookLogRequest {
  string provider_id = 1;
  string webhook_id = 2;  // From payload (idempotency)
  string payload = 3;  // JSON string
  string signature_received = 4;
  bool signature_valid = 5;
  string status = 6;  // "pending", "processing", "success", "failed"
}

message WebhookLog {
  string id = 1;
  string provider_id = 2;
  string webhook_id = 3;
  string status = 4;
  int32 retry_count = 5;
  string error_message = 6;  // Nullable
  string created_at = 7;
  string processed_at = 8;  // Nullable
}
```

**Usage**:

```python
# Webhook service: Log received webhook
log = django_client.CreateWebhookLog(
    provider_id=provider.id,
    webhook_id=payload['id'],
    payload=json.dumps(payload),
    signature_received=signature,
    signature_valid=True,
    status="pending"
)
```

#### GetWebhookLog

```protobuf
rpc GetWebhookLog(GetWebhookLogRequest) returns (WebhookLog) {}

message GetWebhookLogRequest {
  string id = 1;  // UUID
}
```

#### ListWebhookLogs

```protobuf
rpc ListWebhookLogs(ListWebhookLogsRequest) returns (WebhookLogList) {}

message ListWebhookLogsRequest {
  optional string provider_id = 1;
  optional string organization_id = 2;
  optional string status = 3;  // Filter by status
  int32 page = 4;
  int32 page_size = 5;
  optional string start_date = 6;  // ISO8601
  optional string end_date = 7;
}

message WebhookLogList {
  repeated WebhookLog logs = 1;
  int32 total_count = 2;
}
```

#### UpdateWebhookLogStatus

```protobuf
rpc UpdateWebhookLogStatus(UpdateWebhookLogStatusRequest) returns (WebhookLog) {}

message UpdateWebhookLogStatusRequest {
  string id = 1;
  string status = 2;  // "processing", "success", "failed"
  optional string error_message = 3;
}
```

**Usage**:

```python
# Celery task: Mark as success after processing
django_client.UpdateWebhookLogStatus(
    id=log.id,
    status="success"
)
```

---

### 3.5 Webhook Retries

#### CreateWebhookRetry

```protobuf
rpc CreateWebhookRetry(CreateWebhookRetryRequest) returns (WebhookRetry) {}

message CreateWebhookRetryRequest {
  string webhook_log_id = 1;
  int32 attempt_number = 2;
  string error_details = 3;
}

message WebhookRetry {
  string id = 1;
  string webhook_log_id = 2;
  int32 attempt_number = 3;
  string error_details = 4;
  string created_at = 5;  // When retry happened
}
```

#### ListWebhookRetries

```protobuf
rpc ListWebhookRetries(ListWebhookRetriesRequest) returns (WebhookRetryList) {}

message ListWebhookRetriesRequest {
  string webhook_log_id = 1;
}

message WebhookRetryList {
  repeated WebhookRetry retries = 1;
}
```

---

### 3.6 Audit Logs

#### CreateAuditLog

```protobuf
rpc CreateAuditLog(CreateAuditLogRequest) returns (AuditLog) {}

message CreateAuditLogRequest {
  string user_id = 1;
  string action = 2;  // "created", "updated", "deleted"
  string resource_type = 3;  // "organization", "webhook_provider"
  string resource_id = 4;
  string changes = 5;  // JSON string
}

message AuditLog {
  string id = 1;
  string user_id = 2;
  string action = 3;
  string resource_type = 4;
  string resource_id = 5;
  string changes = 6;
  string created_at = 7;
}
```

**Usage**:

```python
# Auth service: Log user login
django_client.CreateAuditLog(
    user_id=user.id,
    action="login",
    resource_type="user",
    resource_id=user.id,
    changes=json.dumps({"ip": request.client.host})
)
```

#### ListAuditLogs

```protobuf
rpc ListAuditLogs(ListAuditLogsRequest) returns (AuditLogList) {}

message ListAuditLogsRequest {
  optional string user_id = 1;
  optional string resource_type = 2;
  optional string resource_id = 3;
  int32 page = 4;
  int32 page_size = 5;
}

message AuditLogList {
  repeated AuditLog logs = 1;
  int32 total_count = 2;
}
```

---

## 4. Auth Service (REST Public)

**Base URL**: `https://api.yourapp.com/auth`

### 4.1 Request Magic Link

```http
POST /auth/request-magic-link
Content-Type: application/json

{
  "email": "john@acme.com"
}

Response 200:
{
  "success": true,
  "message": "Magic link sent to email"
}

Response 404:
{
  "error": "User not found"
}

Response 429:
{
  "error": "Too many requests. Try again in 60 seconds."
}
```

**Flow**:

1. Validate email format
2. Call Django gRPC: `GetUserByEmail(email)`
3. Generate token (save to Redis with 15min TTL)
4. Send email with link: `https://app.com/auth/verify?token=abc123`
5. Return success

### 4.2 Verify Magic Link

```http
GET /auth/verify?token=abc123

Response 200:
{
  "success": true,
  "token": "jwt_token_here",
  "user": {
    "id": "uuid",
    "email": "john@acme.com",
    "organization": "acme-corp",
    "role": "admin"
  }
}

Response 401:
{
  "error": "Invalid or expired token"
}
```

**Flow**:

1. Check Redis for token
2. If valid, get email from Redis
3. Call Django gRPC: `GetUserByEmail(email)`
4. Call Django gRPC: `UpdateLastLogin(user_id)`
5. Generate JWT token
6. Delete token from Redis (one-time use)
7. Return JWT + user info

### 4.3 Logout

```http
POST /auth/logout
Authorization: Bearer {jwt_token}

Response 200:
{
  "success": true
}
```

**Flow**:

1. Validate JWT
2. Add JWT to Redis blacklist (with TTL = token expiry)
3. Return success

---

## 5. Webhook Service (REST Public)

**Base URL**: `https://api.yourapp.com/webhooks`

### 5.1 Receive Webhook

```http
POST /webhooks/{provider}
X-Webhook-Signature: sha256=abc123...
Content-Type: application/json

{
  "id": "evt_123",
  "type": "payment.success",
  "data": {...}
}

Response 200 (Success):
{
  "status": "received",
  "webhook_id": "uuid",
  "message": "Webhook accepted for processing"
}

Response 401 (Invalid Signature):
{
  "error": "Invalid signature"
}

Response 404 (Provider Not Found):
{
  "error": "Provider not configured"
}

Response 503 (Service Unavailable):
{
  "error": "System temporarily unavailable",
  "retry_after": 60
}
```

**Flow**:

1. Extract provider from path (`stripe`, `github`, etc.)
2. Parse `X-Webhook-Signature` header
3. Call Django gRPC: `GetProviderSecret(organization_slug, provider_name)`
   - **Note**: Need organization from request (subdomain or header)
4. Validate HMAC signature
5. Check idempotency: Call Django gRPC: `GetWebhookLog(webhook_id=payload['id'])`
6. If duplicate, return 200 (already processed)
7. Call Django gRPC: `CreateWebhookLog(...)`
8. Dispatch Celery task
9. Return 200 OK within 3 seconds

### 5.2 Test Webhook (Development Only)

```http
POST /webhooks/test/stripe
Authorization: Bearer {jwt_token}

{
  "event_type": "payment.success",
  "amount": 5000
}

Response 200:
{
  "success": true,
  "webhook_id": "uuid",
  "signature": "sha256=abc123..."
}
```

**Purpose**: Generate mock webhook with valid signature for testing

---

## 6. Error Handling Standards

### 6.1 HTTP Status Codes

| Code | Usage          | Example                |
| ---- | -------------- | ---------------------- |
| 200  | Success        | Webhook received       |
| 201  | Created        | Organization created   |
| 400  | Bad Request    | Invalid JSON payload   |
| 401  | Unauthorized   | Invalid signature      |
| 403  | Forbidden      | Inactive organization  |
| 404  | Not Found      | Provider doesn't exist |
| 429  | Rate Limited   | Too many requests      |
| 500  | Internal Error | Unhandled exception    |
| 503  | Unavailable    | Database down          |

### 6.2 Error Response Format

```json
{
  "error": "Human-readable message",
  "code": "INVALID_SIGNATURE",
  "details": {
    "expected": "sha256=...",
    "received": "sha256=..."
  },
  "request_id": "uuid"
}
```

### 6.3 gRPC Error Codes

| Code              | Usage                  |
| ----------------- | ---------------------- |
| OK                | Success                |
| NOT_FOUND         | Resource doesn't exist |
| ALREADY_EXISTS    | Duplicate resource     |
| PERMISSION_DENIED | Unauthorized           |
| INVALID_ARGUMENT  | Bad input              |
| INTERNAL          | Server error           |

---

## 7. Authentication & Authorization

### 7.1 External APIs (Auth/Webhook Services)

**JWT Token Structure**:

```json
{
  "user_id": "uuid",
  "email": "john@acme.com",
  "organization_id": "uuid",
  "role": "admin",
  "exp": 1699999999
}
```

**Authorization Header**:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### 7.2 Internal APIs (gRPC)

**mTLS (Mutual TLS)**:

- Django service certificate
- FastAPI services have client certificates
- Certificate-based authentication (no tokens needed)

**Alternative (Simpler for MVP)**:

- Shared API key in environment variable
- Passed in gRPC metadata

```python
# FastAPI calls Django
metadata = [('api-key', os.getenv('INTERNAL_API_KEY'))]
response = stub.GetOrganization(request, metadata=metadata)
```

---

## 8. API Documentation Standards

### 8.1 How API Documentation is Done Professionally

#### For REST APIs:

1. **OpenAPI/Swagger** (Auto-generated from FastAPI)

   - Hosted at `/docs` endpoint
   - Interactive testing interface
   - Example: `https://api.yourapp.com/docs`

2. **Postman Collections**

   - Export from Swagger
   - Share with team for manual testing

3. **README.md**
   - Quick start guide
   - Authentication examples
   - Common use cases

#### For gRPC APIs:

1. **Protocol Buffer Files** (`.proto`)

   - Self-documenting schemas
   - Committed to version control

2. **gRPC Reflection**

   - Enable reflection in Django service
   - Tools like `grpcurl` can discover endpoints

3. **Generated Markdown**
   - Use `protoc-gen-doc` to generate docs from `.proto` files

### 8.2 Documentation Structure

```
docs/
├── API_SPECIFICATION.md       ← This document
├── API_QUICKSTART.md          ← Getting started guide
├── AUTHENTICATION.md          ← Auth flow details
├── WEBHOOKS.md                ← Webhook integration guide
├── grpc/
│   ├── protos/                ← .proto files
│   │   ├── organization.proto
│   │   ├── user.proto
│   │   └── webhook.proto
│   └── generated-docs/        ← Auto-generated from protos
└── postman/
    ├── auth-service.json
    └── webhook-service.json
```

### 8.3 API Changelog

Track breaking changes:

```markdown
# API Changelog

## v1.1.0 (2025-12-01)

### Added

- `ListOrganizations` endpoint with pagination
- `UpdateLastLogin` for auth tracking

### Changed

- `GetProviderSecret` now requires `organization_slug` (breaking)

### Deprecated

- `GetProvider` without organization filter (use `GetProviderByOrgAndName`)

## v1.0.0 (2025-11-17)

- Initial API release
```

---

## 9. Implementation Checklist

### 9.1 Django Service (gRPC Server)

- [ ] Install `grpcio` and `grpcio-tools`
- [ ] Define `.proto` files
- [ ] Generate Python code from protos
- [ ] Implement gRPC servicers (handlers)
- [ ] Add authentication middleware (API key validation)
- [ ] Enable gRPC reflection (for debugging)
- [ ] Test with `grpcurl`

### 9.2 FastAPI Services (gRPC Clients)

- [ ] Generate gRPC client stubs
- [ ] Create gRPC client wrapper classes
- [ ] Implement connection pooling
- [ ] Add retry logic (transient failures)
- [ ] Mock gRPC calls in tests

### 9.3 Documentation

- [ ] Export OpenAPI spec from FastAPI
- [ ] Generate docs from `.proto` files
- [ ] Create Postman collection
- [ ] Write API quickstart guide
- [ ] Document authentication flow

---

## 10. Next Steps

### Phase 3.3.1: Django gRPC Server Setup

1. **Install dependencies**
2. **Create `.proto` files** (start with Organization)
3. **Generate Python code**
4. **Implement first servicer** (`GetOrganization`)
5. **Test with grpcurl**

### Phase 3.3.2: FastAPI gRPC Client Setup

1. **Generate client stubs**
2. **Create client wrapper**
3. **Test connection to Django**
4. **Implement first endpoint** (webhook receiver)

**Ready to start Phase 3.3.1?** Let me know and I'll guide you through Django gRPC setup!
