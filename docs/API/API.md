# API Specification

This document defines the API surface across Django (core), Auth microservice, and Webhook microservice.
Endpoints are marked as REST (external/public) or gRPC (internal service-to-service).

---

## 1. Django Service (Core DB + Admin)

### 1.1 Organizations

- gRPC GET /organizations/{slug} → Org details
- gRPC POST /organizations → Create new org
- gRPC PATCH /organizations/{slug} → Update plan, webhook limit
- gRPC GET /organizations/{slug}/providers → List providers for org

### 1.2 Users

- gRPC GET /users/{id} → Fetch user details
- gRPC GET /users?organization={slug} → List users in org
- gRPC PATCH /users/{id} → Update role/status
- gRPC GetUser(UserRequest) → Internal lookup for Auth service

### 1.3 Webhook Providers

- gRPC POST /providers → Create provider (with secret)
- gRPC GET /providers/{id} → Provider details (secret excluded)
- gRPC PATCH /providers/{id} → Update provider settings
- REST DELETE /providers/{id} → Remove provider
- gRPC GET /providers/{id}/secret → Internal endpoint for FastAPI
- gRPC GetProviderSecret(ProviderRequest) → Internal call for webhook validation

### 1.4 Webhook Logs

- gRPC GET /webhooks/{id} → Fetch webhook log details
- gRPC GET /webhooks?provider={id} → List logs for provider
- gRPC PATCH /webhooks/{id} → Update status (manual fix)

### 1.5 Webhook Retries

- gRPC GET /webhooks/{id}/retries → List retries
- gRPC POST /webhooks/{id}/retries → Trigger retry attempt

### 1.6 Audit Logs

- gRPC GET /audit?user={id} → User activity history
- gRPC GET /audit?resource={type}&id={uuid} → Resource history
- gRPC GET /audit → General audit log listing (read-only)

---

## 2. Auth Microservice (FastAPI)

### 2.1 Authentication

- REST POST /auth/login → Request passwordless login (magic link)
- REST POST /auth/verify → Verify magic link token
- REST POST /auth/logout → Invalidate session/token

### 2.2 Integration with Django

- gRPC GetUser(UserRequest) → Fetch user details from Django
- gRPC GetOrganization(OrgRequest) → Fetch org info for auth context

---

## 3. Webhook Microservice (FastAPI)

### 3.1 Webhook Receiver

- REST POST /webhooks/{provider} → Receive webhook event
  - Validates HMAC-SHA256 signature
  - Returns 200 OK within 3s (timeout compliance)
  - Dispatches background task (Celery)

### 3.2 Integration with Django

- gRPC GetProviderSecret(ProviderRequest) → Fetch secret for signature validation
- gRPC LogWebhook(LogRequest) → Record webhook log in Django
- gRPC LogRetry(RetryRequest) → Record retry attempt in Django

### 3.3 Testing

- REST POST /webhooks/test/stripe → Mock Stripe webhook for integration testing
