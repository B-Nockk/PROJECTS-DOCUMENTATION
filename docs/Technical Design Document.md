# Custom Third-Party Webhook Processor

## Technical Design Document (TDD)

> **Document Type**: Technical Design Document (TDD)

> **Project Phase**: Implementation

> **Last Updated**: November 13, 2025

> **Author**: Benson OgheneOchuko

---

## 1. Executive Summary

### 1.1 Project Overview

A production-grade webhook processing system that demonstrates enterprise-level integration patterns. The system securely receives, validates, and processes webhooks from third-party services (Stripe, GitHub, payment providers) with fault tolerance and manual recovery capabilities.

### 1.2 Business Value

- **For Employers**: Demonstrates understanding of real-world integration challenges, security, and operational resilience
- **For Startups**: Production-ready infrastructure for any API-first business
- **Market Fit**: Direct applicability to fintech, e-commerce, and SaaS platforms

### 1.3 Tech Stack

- **Backend (Public API)**: FastAPI (Python 3.11+)
- **Backend (Admin/Config)**: Django 5.0+ with Django Admin
- **Frontend (Demo Portal)**: HTML/CSS/JavaScript (Single-page testing interface)
- **Database**: PostgreSQL
- **Task Queue**: Celery + Redis
- **Deployment**: Docker Compose

---

## 2. System Architecture

### 2.1 High-Level Architecture

```
┌─────────────────┐
│  Third-Party    │
│  Services       │
│  (Stripe, etc)  │
└────────┬────────┘
         │ HTTPS POST
         ▼
┌─────────────────────────────────────────────┐
│           FastAPI Layer (Public)            │
│  ┌─────────────────────────────────────┐   │
│  │ POST /webhooks/{provider}           │   │
│  │ - Signature Validation (HMAC-SHA256)│   │
│  │ - Immediate 200 OK Response         │   │
│  │ - Background Task Dispatch          │   │
│  └─────────────────────────────────────┘   │
└─────────────────┬───────────────────────────┘
                  │
                  ▼
         ┌────────────────┐
         │ Celery Worker  │
         │ (Async Processing)
         └────────┬───────┘
                  │
                  ▼
         ┌────────────────────┐
         │   PostgreSQL DB    │
         │  - Webhook Logs    │
         │  - Retry History   │
         │  - Provider Configs│
         └────────────────────┘
                  ▲
                  │
┌─────────────────┴───────────────────────────┐
│        Django Admin Layer (Internal)        │
│  ┌─────────────────────────────────────┐   │
│  │ - Manage Webhook Secrets            │   │
│  │ - View Webhook Logs                 │   │
│  │ - Manual Retry Interface            │   │
│  │ - Configuration Dashboard           │   │
│  └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
                  ▲
                  │
         ┌────────┴─────────┐
         │  Demo Portal     │
         │  (Testing UI)    │
         │  - Send Test     │
         │  - View Response │
         └──────────────────┘
```

### 2.2 Component Responsibilities

#### FastAPI Service (Port 8000)

- **Webhook Receiver Endpoint**: `POST /webhooks/{provider}`
- **Signature Validation**: HMAC-SHA256 verification
- **Response Time**: <3 seconds (webhook timeout compliance)
- **Background Tasks**: Celery task dispatch

#### Django Service (Port 8001)

- **Admin Interface**: Manage configurations at `/admin`
- **Webhook Secrets CRUD**: Store provider API keys/secrets
- **Log Viewer**: Display all received webhooks with status
- **Retry Interface**: Manual re-processing button for failed webhooks

#### Celery Workers

- **Async Processing**: Handle webhook payload after acknowledgment
- **Retry Logic**: Exponential backoff (3 attempts: 1min, 5min, 15min)
- **Dead Letter Queue**: Failed webhooks after max retries

#### Demo Portal (Static HTML)

- **Webhook Simulator**: Form to send test webhooks
- **Response Viewer**: Display FastAPI responses in real-time
- **Log Inspector**: Fetch and display webhook logs from Django API

---

## 3. Data Models

### 3.1 WebhookProvider (Django Model)

```python
class WebhookProvider(models.Model):
    name = models.CharField(max_length=100)  # e.g., "stripe", "github"
    secret_key = models.CharField(max_length=255)  # HMAC secret
    is_active = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
```

### 3.2 WebhookLog (Django Model)

```python
class WebhookLog(models.Model):
    STATUS_CHOICES = [
        ('pending', 'Pending'),
        ('processing', 'Processing'),
        ('success', 'Success'),
        ('failed', 'Failed'),
    ]

    provider = models.ForeignKey(WebhookProvider, on_delete=models.CASCADE)
    webhook_id = models.UUIDField(unique=True)  # Idempotency key
    payload = models.JSONField()
    signature_received = models.CharField(max_length=255)
    signature_valid = models.BooleanField()
    status = models.CharField(max_length=20, choices=STATUS_CHOICES)
    retry_count = models.IntegerField(default=0)
    error_message = models.TextField(null=True, blank=True)
    received_at = models.DateTimeField(auto_now_add=True)
    processed_at = models.DateTimeField(null=True, blank=True)
```

---

## 4. Security Implementation

### 4.1 HMAC-SHA256 Signature Validation

```python
import hmac
import hashlib

def verify_signature(payload: bytes, signature: str, secret: str) -> bool:
    """
    Validates webhook signature using HMAC-SHA256.

    Args:
        payload: Raw request body (bytes)
        signature: Signature from request header
        secret: Provider secret key from database

    Returns:
        True if signature is valid, False otherwise
    """
    expected_signature = hmac.new(
        key=secret.encode(),
        msg=payload,
        digestmod=hashlib.sha256
    ).hexdigest()

    return hmac.compare_digest(expected_signature, signature)
```

### 4.2 Request Headers Expected

- **Stripe**: `Stripe-Signature` header
- **GitHub**: `X-Hub-Signature-256` header
- **Generic**: `X-Webhook-Signature` header

---

## 5. API Specifications

### 5.1 FastAPI Endpoints

#### Receive Webhook

```
POST /webhooks/{provider}
Headers:
  - X-Webhook-Signature: <hmac_signature>
  - Content-Type: application/json

Body:
{
  "event": "payment.success",
  "data": {
    "order_id": "12345",
    "amount": 5000
  }
}

Response (200 OK):
{
  "status": "received",
  "webhook_id": "uuid-here",
  "message": "Webhook accepted for processing"
}

Response (401 Unauthorized):
{
  "detail": "Invalid signature"
}
```

#### Health Check

```
GET /health

Response (200 OK):
{
  "status": "healthy",
  "timestamp": "2025-11-13T10:30:00Z"
}
```

### 5.2 Django Admin Custom Actions

- **Retry Failed Webhook**: Button in admin list view
- **View Payload**: Modal to inspect full JSON payload
- **Signature Debugger**: Compare expected vs received signatures

---

## 6. Processing Flow

### 6.1 Happy Path

```
1. Third-party sends POST to /webhooks/stripe
2. FastAPI extracts signature from header
3. Fetch provider secret from Django DB (via API call)
4. Validate signature using HMAC-SHA256
5. Return 200 OK immediately (within 500ms)
6. Dispatch Celery task with webhook data
7. Celery worker processes payload
8. Update WebhookLog status to 'success'
```

### 6.2 Failure Path (Invalid Signature)

```
1-3. (Same as above)
4. Signature validation fails
5. Log attempt to database with status='failed'
6. Return 401 Unauthorized
7. Admin can view failed attempt in Django Admin
```

### 6.3 Retry Logic

```
1. Celery task fails during processing
2. Log error_message to WebhookLog
3. Retry after 1 minute (retry_count=1)
4. If fails again, retry after 5 minutes (retry_count=2)
5. If fails again, retry after 15 minutes (retry_count=3)
6. After 3 failures, move to Dead Letter Queue
7. Admin sees "Manual Retry" button in Django Admin
```

---

## 7. Demo Portal Specifications

### 7.1 Purpose

A **minimal but functional** testing interface that demonstrates the webhook system without requiring a separate frontend project. This is NOT a production admin interface—it's a developer tool for showcasing the project.

### 7.2 Features

1. **Webhook Simulator**

   - Dropdown to select provider (Stripe, GitHub, PayPal)
   - JSON editor for custom payload
   - "Send Webhook" button that generates HMAC signature
   - Real-time response display

2. **Log Viewer**

   - Table showing recent webhooks (last 20)
   - Columns: Timestamp, Provider, Status, Retry Count
   - Click row to view full payload in modal

3. **Configuration Panel**
   - Display active providers and their status
   - Button to "Add New Provider" (opens Django Admin)

### 7.3 Technology

- **Single HTML file** with embedded CSS/JavaScript
- **No framework** (vanilla JS for simplicity)
- **Fetch API** for AJAX calls to FastAPI/Django
- **Responsive design** (works on mobile for demos)

### 7.4 User Flow

```
Developer opens portal → Selects "Stripe" provider →
Enters test payload → Clicks "Send" →
Sees "200 OK" response → Refreshes log viewer →
Sees webhook in "processing" status →
After 2 seconds, status changes to "success"
```

---

## 8. Development Phases

### Phase 1: Foundation (Week 1)

- [ ] Setup project structure (monorepo with Django/FastAPI)
- [ ] Configure PostgreSQL database
- [ ] Create Django models (WebhookProvider, WebhookLog)
- [ ] Setup Django Admin interface

### Phase 2: FastAPI Core (Week 2)

- [ ] Implement webhook receiver endpoint
- [ ] HMAC signature validation logic
- [ ] Database integration (read provider secrets)
- [ ] Background task dispatch (mock Celery initially)

### Phase 3: Async Processing (Week 3)

- [ ] Setup Celery + Redis
- [ ] Implement webhook processing task
- [ ] Retry logic with exponential backoff
- [ ] Dead letter queue handling

### Phase 4: Demo Portal (Week 4)

- [ ] Build single-page HTML interface
- [ ] Webhook simulator with signature generation
- [ ] Log viewer with real-time updates
- [ ] Integration with FastAPI endpoints

### Phase 5: Production Ready (Week 5)

- [ ] Docker Compose configuration
- [ ] Environment variable management (.env)
- [ ] Logging and monitoring setup
- [ ] Write comprehensive README

### Phase 6: Polish (Week 6)

- [ ] Load testing (simulate 100 webhooks/sec)
- [ ] Security audit (rate limiting, input validation)
- [ ] Documentation (API docs, sequence diagrams)
- [ ] Deploy to cloud (Railway/Render)

---

## 9. Testing Strategy

### 9.1 Unit Tests

- Signature validation function
- HMAC generation
- Idempotency key handling

### 9.2 Integration Tests

- End-to-end webhook flow (send → receive → process)
- Retry mechanism (force failures)
- Admin actions (manual retry)

### 9.3 Load Testing

- 100 webhooks/second for 1 minute
- Measure: Response time, failure rate, database load

---

## 10. Deployment

### 10.1 Local Development

```bash
docker-compose up
# FastAPI: http://localhost:8000
# Django Admin: http://localhost:8001/admin
# Demo Portal: http://localhost:8000/demo
```

### 10.2 Production (Railway/Render)

- FastAPI service (public endpoint)
- Django service (internal, IP-restricted)
- PostgreSQL database
- Redis for Celery
- Environment variables in platform dashboard

---

## 11. Success Metrics

### 11.1 Technical

- [ ] 99.9% signature validation accuracy
- [ ] <500ms response time (p95)
- [ ] <1% webhook processing failure rate
- [ ] Retry recovery rate >95%

### 11.2 Portfolio

- [ ] Live demo URL in resume
- [ ] GitHub README with architecture diagram
- [ ] Video walkthrough (3-5 minutes)
- [ ] Blog post explaining design decisions

---

## 12. Key Patterns & Principles

### 12.1 Design Patterns Used

- **Repository Pattern**: Django ORM abstracts database access
- **Background Job Pattern**: Celery for async processing
- **Idempotency**: Webhook IDs prevent duplicate processing
- **Circuit Breaker**: Retry with exponential backoff
- **Admin Interface Pattern**: Django Admin for operations

### 12.2 Best Practices

- **Security First**: Always validate signatures before processing
- **Fast Acknowledgment**: Respond within 3 seconds
- **Immutable Logs**: Never delete webhook logs (audit trail)
- **Graceful Degradation**: Failed webhooks don't block system
- **Configuration as Code**: Docker Compose for reproducibility

### 12.3 Common Pitfalls Avoided

- ❌ Processing in request handler (blocks response)
- ❌ Trusting webhook data without signature validation
- ❌ No retry mechanism for transient failures
- ❌ Exposing admin interface publicly
- ❌ Hardcoding secrets in code

---

## 13. Future Enhancements

### 13.1 Nice-to-Haves (Post-MVP)

- [ ] Webhook versioning support (v1, v2 endpoints)
- [ ] Rate limiting per provider
- [ ] Prometheus metrics export
- [ ] Webhook replay UI (resend from admin)
- [ ] Multi-tenant support (separate secrets per client)

### 13.2 Advanced Features

- [ ] GraphQL API for webhook logs
- [ ] Real-time dashboard with WebSockets
- [ ] Machine learning for fraud detection
- [ ] API key rotation workflow

---

## 14. Documentation Checklist

- [ ] README.md with setup instructions
- [ ] API documentation (OpenAPI/Swagger)
- [ ] Architecture diagram (draw.io or Excalidraw)
- [ ] Sequence diagram for webhook flow
- [ ] Environment variables reference
- [ ] Troubleshooting guide

---

## Appendix A: Professional Document Types Reference

**Documents used in this project:**

1. **Technical Design Document (TDD)** - _This document_

   - Comprehensive technical specifications
   - Architecture decisions and rationale

2. **Product Requirements Document (PRD)**

   - User stories and business requirements
   - Not needed for this solo project

3. **API Specification**

   - OpenAPI/Swagger documentation
   - Auto-generated from FastAPI

4. **README.md**

   - Quick start guide for developers
   - High-level project overview

5. **Architecture Decision Records (ADRs)**
   - Document why specific technologies were chosen
   - Example: "Why FastAPI instead of Flask?"

**Note**: For this project, the TDD serves as our single source of truth. In larger teams, you'd have separate PRDs (product managers), TDDs (engineers), and ADRs (architects).
