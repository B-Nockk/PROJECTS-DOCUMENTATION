# Webhook Processor - Database Schema Document

## Data Models & Business Logic Specification

> **Document Type**: Database Schema Document (DSD) / Entity-Relationship Documentation

> **Version**: 1.0

> **Last Updated**: November 13, 2025

> **Purpose**: Define database structure, relationships, and business rules

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [User Types & Access Levels](#2-user-types--access-levels)
3. [Database Architecture](#3-database-architecture)
4. [Core Models](#4-core-models)
5. [Business Logic Rules](#5-business-logic-rules)
6. [Authentication Strategy](#6-authentication-strategy)
7. [Dashboard Assignment](#7-dashboard-assignment)

---

## 1. System Overview

### 1.1 What This System Does

This is a **multi-tenant webhook processing platform** where:

- **Customers** (businesses) sign up to receive webhook processing services
- Each customer connects their third-party services (Stripe, GitHub, etc.)
- Our system receives webhooks, validates them, and processes them reliably
- Customers view logs, configure settings, and monitor webhook health

### 1.2 Real-World Analogy

Think of it like **a secure mail sorting facility**:

- **Customer** = Company that receives mail
- **Webhook Provider** = Different postal services (FedEx, UPS, DHL)
- **Webhook** = Individual package
- **Our System** = Sorting facility that validates sender, logs packages, handles failures

---

## 2. User Types & Access Levels

### 2.1 User Hierarchy

```
┌─────────────────────────────────────────────┐
│           SUPER ADMIN (You)                 │
│  - Manage all customers                     │
│  - View system-wide analytics               │
│  - Configure platform settings              │
│  - Access Django Admin                      │
└─────────────────┬───────────────────────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
┌───────▼────────┐  ┌──────▼─────────┐
│   CUSTOMER     │  │   CUSTOMER     │
│   (Company A)  │  │   (Company B)  │
│                │  │                │
│  - Org Admin   │  │  - Org Admin   │
│  - Team Member │  │  - Team Member │
└────────────────┘  └────────────────┘
```

### 2.2 Permission Matrix

| Action                | Super Admin | Org Admin  | Team Member |
| --------------------- | ----------- | ---------- | ----------- |
| View own webhooks     | ✅ All      | ✅ Own org | ✅ Own org  |
| Add webhook providers | ✅          | ✅         | ❌          |
| Manage team members   | ✅          | ✅         | ❌          |
| View billing          | ✅ All      | ✅ Own org | ❌          |
| Retry failed webhooks | ✅          | ✅         | ✅          |
| Access Django Admin   | ✅          | ❌         | ❌          |

### 2.3 Why This Matters

- **Business Logic**: Customers should ONLY see their own data (multi-tenancy)
- **Security**: Org admins can't see other organizations' webhook secrets
- **Scalability**: Easy to add new permission levels later

---

## 3. Database Architecture

### 3.1 Entity-Relationship Diagram

```
┌──────────────────┐
│   Organization   │ ← Customer's company
│  - name          │
│  - slug          │
│  - is_active     │
└────────┬─────────┘
         │ 1
         │
         │ N
┌────────▼─────────────────┐
│   User                   │ ← People who login
│  - email                 │
│  - organization_id (FK)  │
│  - role (admin/member)   │
│  - magic_link_token      │ ← For passwordless login
└──────────────────────────┘

┌────────────────────────────┐
│   WebhookProvider          │ ← Customer's integrations
│  - organization_id (FK)    │
│  - provider_name           │
│  - secret_key_encrypted    │
│  - is_active               │
└────────┬───────────────────┘
         │ 1
         │
         │ N
┌────────▼───────────────────┐
│   WebhookLog               │ ← Individual webhook events
│  - provider_id (FK)        │
│  - webhook_id (UUID)       │
│  - payload (JSON)          │
│  - status                  │
│  - signature_valid         │
│  - retry_count             │
│  - error_message           │
└────────────────────────────┘

┌────────────────────────────┐
│   WebhookRetry             │ ← Retry history tracking
│  - webhook_log_id (FK)     │
│  - attempt_number          │
│  - attempted_at            │
│  - error_details           │
└────────────────────────────┘

┌────────────────────────────┐
│   AuditLog                 │ ← Who did what, when
│  - user_id (FK)            │
│  - action                  │
│  - resource_type           │
│  - resource_id             │
│  - changes (JSON)          │
└────────────────────────────┘
```

### 3.2 Why These Tables Exist

#### **Organization Table**

- **Business Need**: Support multiple customers on same platform
- **Real-World**: "Acme Corp" and "TechStart Inc" both use your service
- **Why Not Just Users?**: One company might have 10 team members - they need shared access

#### **User Table**

- **Business Need**: People need to login to view dashboards
- **Real-World**: john@acme.com and sarah@acme.com both work at Acme Corp
- **Why Separate from Organization?**: One org can have many users

#### **WebhookProvider Table**

- **Business Need**: Each customer connects different services
- **Real-World**: Acme Corp uses Stripe + GitHub, TechStart uses only PayPal
- **Why Encrypted Secrets?**: Protect customer data if database is breached

#### **WebhookLog Table**

- **Business Need**: Track every webhook received for debugging/compliance
- **Real-World**: "Why didn't my order webhook process?" - Check the logs
- **Why JSON Payload?**: Flexible structure (Stripe vs GitHub have different formats)

#### **WebhookRetry Table**

- **Business Need**: Track retry attempts separately from main log
- **Real-World**: "This webhook was retried 3 times before succeeding"
- **Why Separate Table?**: One webhook can have 0-3 retry records

#### **AuditLog Table**

- **Business Need**: Compliance and security (who changed what)
- **Real-World**: "Who deleted our Stripe integration at 2am?"
- **Why Needed?**: Legal requirement for SOC 2 compliance, fraud detection

---

## 4. Core Models

### 4.1 Base Model (Your DBBaseModel Pattern)

```python
# django_service/webhooks/models.py

from django.db import models
import uuid
from datetime import datetime

class DBBaseModel(models.Model):
    """
    Base model for all database tables.
    Provides common fields and methods.

    WHY THIS EXISTS:
    - DRY principle: Don't repeat id/timestamps in every model
    - Consistent UUIDs across all tables (better for distributed systems)
    - Common methods (to_dict) available everywhere
    """
    id = models.UUIDField(
        primary_key=True,
        default=uuid.uuid4,
        editable=False,
        help_text="Unique identifier (UUID v4)"
    )
    created_at = models.DateTimeField(
        auto_now_add=True,
        help_text="Timestamp when record was created"
    )
    updated_at = models.DateTimeField(
        auto_now=True,
        help_text="Timestamp when record was last modified"
    )

    class Meta:
        abstract = True  # This won't create a table, only inherited models will
        ordering = ['-created_at']  # Default: newest first

    def to_dict(self):
        """
        Convert model instance to dictionary.
        Child models can override to customize output.
        """
        data = {}
        for field in self._meta.fields:
            value = getattr(self, field.name)

            # Handle different field types
            if isinstance(value, datetime):
                data[field.name] = value.isoformat()
            elif isinstance(value, uuid.UUID):
                data[field.name] = str(value)
            else:
                data[field.name] = value

        return data

    def __str__(self):
        """Default string representation"""
        return f"{self.__class__.__name__}({self.id})"
```

**LEARNING NOTE**:

- `abstract = True` means Django won't create a `DBBaseModel` table
- All child models will have `id`, `created_at`, `updated_at` automatically
- You can override `to_dict()` in child models for custom behavior

---

### 4.2 Organization Model (I'll create, you observe)

```python
class Organization(DBBaseModel):
    """
    Represents a customer company using the webhook service.

    BUSINESS LOGIC:
    - One organization can have many users
    - One organization can have many webhook providers
    - Organizations are isolated (can't see each other's data)

    REAL-WORLD EXAMPLE:
    - name: "Acme Corporation"
    - slug: "acme-corp" (used in URLs: /dashboard/acme-corp)
    - plan: "premium" (for billing)
    """
    name = models.CharField(
        max_length=255,
        help_text="Company name (e.g., 'Acme Corporation')"
    )
    slug = models.SlugField(
        unique=True,
        max_length=100,
        help_text="URL-friendly identifier (e.g., 'acme-corp')"
    )
    plan = models.CharField(
        max_length=20,
        choices=[
            ('free', 'Free'),
            ('starter', 'Starter'),
            ('premium', 'Premium'),
        ],
        default='free',
        help_text="Billing plan tier"
    )
    webhook_limit = models.IntegerField(
        default=1000,
        help_text="Max webhooks per month"
    )
    is_active = models.BooleanField(
        default=True,
        help_text="Organization is active and can receive webhooks"
    )

    class Meta:
        db_table = 'organizations'
        verbose_name = 'Organization'
        verbose_name_plural = 'Organizations'

    def __str__(self):
        return self.name

    def to_dict(self):
        """Override to add computed fields"""
        data = super().to_dict()
        data['webhook_count'] = self.webhooklog_set.count()  # Total webhooks received
        return data
```

**WHY EACH FIELD EXISTS**:

- `slug`: Used in URLs like `/dashboard/acme-corp/webhooks`
- `plan`: Determines features (free = 1000 webhooks/month, premium = unlimited)
- `webhook_limit`: Business rule - prevent abuse/manage costs
- `is_active`: Soft delete (don't actually delete data, just deactivate)

---

### 4.3 User Model (YOUR TURN - I'll give structure, you implement)

**ASSIGNMENT FOR YOU**:
Create the `User` model following this specification:

**Requirements**:

1. Inherit from `DBBaseModel`
2. Add these fields:

   - `email` (unique, used for login)
   - `organization` (ForeignKey to Organization)
   - `role` (choices: 'admin', 'member')
   - `magic_link_token` (for passwordless login, can be null)
   - `token_expires_at` (datetime, can be null)
   - `is_active` (boolean, default True)
   - `last_login_at` (datetime, can be null)

3. Add a method: `generate_magic_link()` that creates a token
4. Override `__str__` to return email

**Hints**:

```python
class User(DBBaseModel):
    """
    Represents a person who can login to the dashboard.

    BUSINESS LOGIC:
    - Each user belongs to ONE organization
    - Users can be 'admin' (manage settings) or 'member' (view only)
    - Passwordless login using magic links sent via email

    TODO: Implement this model following Organization pattern above
    """
    # YOUR CODE HERE
    pass
```

**I'll review your implementation and we'll discuss it!**

---

### 4.4 WebhookProvider Model (I'll create - encryption example)

```python
from cryptography.fernet import Fernet
from django.conf import settings

class WebhookProvider(DBBaseModel):
    """
    Stores third-party service configurations per organization.

    BUSINESS LOGIC:
    - Each org can have one provider of each type (e.g., one Stripe account)
    - Secrets are encrypted at rest (not plain text in database)
    - When deleted, webhooks from this provider are rejected

    SECURITY:
    - secret_key is NEVER returned in API responses
    - Only decrypted in memory during webhook validation
    """
    organization = models.ForeignKey(
        Organization,
        on_delete=models.CASCADE,  # Delete providers if org is deleted
        related_name='webhook_providers',
        help_text="Organization that owns this provider"
    )
    provider_name = models.CharField(
        max_length=50,
        choices=[
            ('stripe', 'Stripe'),
            ('github', 'GitHub'),
            ('paypal', 'PayPal'),
            ('shopify', 'Shopify'),
        ],
        help_text="Type of webhook provider"
    )
    secret_key_encrypted = models.BinaryField(
        help_text="Encrypted webhook signing secret"
    )
    is_active = models.BooleanField(
        default=True,
        help_text="Provider is accepting webhooks"
    )

    class Meta:
        db_table = 'webhook_providers'
        unique_together = ['organization', 'provider_name']  # One Stripe per org
        verbose_name = 'Webhook Provider'

    def set_secret(self, plain_secret: str):
        """
        Encrypt and store webhook secret.

        SECURITY NOTE:
        - Uses Fernet (AES-128-CBC with HMAC)
        - Master key stored in environment variable
        - If master key is lost, secrets cannot be recovered
        """
        f = Fernet(settings.SECRET_ENCRYPTION_KEY.encode())
        self.secret_key_encrypted = f.encrypt(plain_secret.encode())

    def get_secret(self) -> str:
        """
        Decrypt and return webhook secret.

        ONLY call this when validating webhooks!
        Never log or expose the decrypted secret.
        """
        f = Fernet(settings.SECRET_ENCRYPTION_KEY.encode())
        return f.decrypt(self.secret_key_encrypted).decode()

    def __str__(self):
        return f"{self.organization.name} - {self.provider_name}"

    def to_dict(self):
        """Override to EXCLUDE secret_key"""
        data = super().to_dict()
        del data['secret_key_encrypted']  # Never expose secrets!
        data['has_secret'] = True  # Just indicate it exists
        return data
```

**SECURITY LESSONS**:

- `set_secret()` / `get_secret()` pattern keeps encryption logic in one place
- `to_dict()` override prevents accidental secret exposure in APIs
- `unique_together` prevents duplicate providers per org

---

### 4.5 WebhookLog Model (YOUR TURN - Practice)

**ASSIGNMENT FOR YOU**:
Create `WebhookLog` model with these requirements:

**Fields**:

- Inherits from `DBBaseModel`
- `provider` (ForeignKey to WebhookProvider)
- `webhook_id` (UUIDField, unique - prevents duplicate processing)
- `payload` (JSONField - stores the actual webhook data)
- `signature_received` (CharField, max 255)
- `signature_valid` (BooleanField)
- `status` (CharField with choices: 'pending', 'processing', 'success', 'failed')
- `retry_count` (IntegerField, default 0)
- `error_message` (TextField, can be null)
- `processed_at` (DateTimeField, can be null)

**Methods to implement**:

- `mark_success()` - Sets status='success', processed_at=now
- `mark_failed(error_msg)` - Sets status='failed', saves error
- `can_retry()` - Returns True if retry_count < 3

**Meta options**:

- `db_table = 'webhook_logs'`
- `ordering = ['-created_at']`
- Add index on `webhook_id` (for fast lookups)

**Start coding and I'll review!**

---

### 4.6 Additional Models (We'll create together later)

#### WebhookRetry

```python
class WebhookRetry(DBBaseModel):
    """Tracks individual retry attempts"""
    webhook_log = models.ForeignKey(WebhookLog, on_delete=models.CASCADE)
    attempt_number = models.IntegerField()
    error_details = models.TextField()
    # attempted_at is already in DBBaseModel as created_at
```

#### AuditLog

```python
class AuditLog(DBBaseModel):
    """Tracks user actions for compliance"""
    user = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)
    action = models.CharField(max_length=50)  # 'created', 'deleted', 'updated'
    resource_type = models.CharField(max_length=50)  # 'webhook_provider', 'user'
    resource_id = models.UUIDField()
    changes = models.JSONField()  # {"old": {...}, "new": {...}}
```

---

## 5. Business Logic Rules

### 5.1 Multi-Tenancy (Data Isolation)

**Rule**: Users can ONLY access data from their organization.

**Implementation**:

```python
# GOOD: Query filters by organization
webhooks = WebhookLog.objects.filter(
    provider__organization=request.user.organization
)

# BAD: Returns all webhooks (security breach!)
webhooks = WebhookLog.objects.all()
```

### 5.2 Webhook Processing State Machine

```
[Received] → [Pending] → [Processing] → [Success]
                                      ↓
                                  [Failed] → [Retry] → [Processing]
                                                     ↓ (after 3 failures)
                                                 [Dead Letter Queue]
```

**Rules**:

- Webhook stays in `pending` for max 30 seconds (timeout protection)
- Retry delays: 1 min → 5 min → 15 min
- After 3 failures, manual intervention required

### 5.3 Provider Activation Rules

**Rule**: Inactive providers reject webhooks immediately.

**Why**: Customer disabled integration temporarily (e.g., testing)

**Implementation**:

```python
if not provider.is_active:
    return {"error": "Provider disabled"}, 403
```

---

## 6. Authentication Strategy

### 6.1 Passwordless Login Flow

```
1. User enters email → POST /api/auth/request-magic-link
2. System generates token → Saves to User.magic_link_token
3. Email sent with link: https://app.com/auth/verify?token=abc123
4. User clicks link → POST /api/auth/verify-magic-link
5. System validates token → Returns JWT session token
6. Frontend stores JWT → Uses for authenticated requests
```

### 6.2 Why Passwordless?

**User Experience**:

- ✅ No passwords to remember
- ✅ Faster login (click email link)
- ✅ No password reset flows needed

**Security**:

- ✅ No password database breaches
- ✅ Phishing harder (token expires in 15 min)
- ✅ No weak passwords

**Implementation Later**: We'll add this in Phase 4 after core webhook logic works.

---

## 7. Dashboard Assignment (Django vs FastAPI)

### 7.1 Architecture Decision

**RECOMMENDATION**: Hybrid approach

```
┌─────────────────────────────────────────────┐
│           Django (Admin Portal)             │
│  - Super Admin Dashboard                    │
│  - Organization Management                  │
│  - User Management                          │
│  - System Settings                          │
│  WHY: Django Admin is perfect for CRUD      │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│         FastAPI (Customer Dashboard)        │
│  - Webhook Logs (real-time, needs speed)   │
│  - Analytics Dashboard (async queries)      │
│  - Webhook Testing Interface                │
│  WHY: Async = faster for high-volume reads  │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│         Demo Portal (Static HTML)           │
│  - Public-facing testing interface          │
│  - No auth required (for demos)             │
│  WHY: Simple, fast, easy to share           │
└─────────────────────────────────────────────┘
```

### 7.2 Decision Matrix

| Feature             | Django | FastAPI | Reasoning                      |
| ------------------- | ------ | ------- | ------------------------------ |
| Super admin CRUD    | ✅     | ❌      | Django Admin is built for this |
| Webhook logs (read) | ❌     | ✅      | FastAPI async queries faster   |
| User management     | ✅     | ❌      | Django has built-in auth       |
| Real-time analytics | ❌     | ✅      | WebSockets/async needed        |
| Retry button        | ❌     | ✅      | Triggers Celery task (async)   |

### 7.3 Your Question: "Don't admins need async too?"

**Short Answer**: Not really for admin tasks!

**Why**:

- Admin actions are **low frequency** (configure provider once per month)
- Webhook logs are **high frequency** (thousands per minute)
- Admin can wait 200ms, webhook viewer needs <50ms response

**Example**:

```python
# Admin: Create provider (happens once)
# Django: Takes 100ms → Acceptable ✅

# Customer: View last 1000 webhooks (happens constantly)
# Django: Takes 500ms → Too slow ❌
# FastAPI async: Takes 50ms → Perfect ✅
```

---

## 8. Implementation Checklist

### Phase 1: Models (This Week)

- [ ] Create `DBBaseModel` (I provided)
- [ ] Create `Organization` (I provided)
- [ ] Create `User` (YOU implement)
- [ ] Create `WebhookProvider` (I provided with encryption)
- [ ] Create `WebhookLog` (YOU implement)
- [ ] Create `WebhookRetry` (We'll do together)
- [ ] Create `AuditLog` (We'll do together)

### Phase 2: Django Setup

- [ ] Register models in admin.py
- [ ] Create admin customizations (filters, search)
- [ ] Add custom admin actions (retry webhook button)
- [ ] Test encryption (add provider, verify secret encrypted)

### Phase 3: FastAPI Integration

- [ ] Create Django REST API endpoints
- [ ] FastAPI calls Django to get secrets
- [ ] Implement webhook receiver
- [ ] Test end-to-end flow

---

## 9. Learning Resources

### 9.1 Key Django Concepts You'll Learn

- **Models**: Object-relational mapping (ORM)
- **Migrations**: Database schema versioning
- **Admin**: Auto-generated CRUD interface
- **Signals**: Trigger actions on model events (e.g., log when provider deleted)

### 9.2 Key Patterns in This Schema

- **Abstract Base Classes**: DBBaseModel (DRY principle)
- **Encryption at Rest**: Fernet for secret storage
- **Soft Deletes**: is_active instead of DELETE
- **Audit Trails**: AuditLog for compliance
- **Multi-Tenancy**: Organization FK on everything

---

## 10. Next Steps

**After you implement User and WebhookLog models:**

1. Run migrations: `python manage.py makemigrations`
2. Apply to database: `python manage.py migrate`
3. Register in admin: Edit `webhooks/admin.py`
4. Test in Django Admin: Add an organization, add a user

**Then we'll**:

- Add encryption key generation script
- Create sample data (seed script)
- Build Django REST API endpoints
- Connect FastAPI to Django

---

## Appendix: Professional Document Names

This document is called a **Database Schema Document** or **Data Model Specification**.

Other names you might see:

- **ERD (Entity-Relationship Diagram)** - Visual diagram of tables
- **Data Dictionary** - Detailed field descriptions
- **Schema Design Document** - Same as this
- **Domain Model** - Focuses on business logic vs. database structure

In large teams:

- **Data Architects** create this document
- **Backend Engineers** implement the models
- **DBAs** optimize indexes and queries
- **Product Managers** define business rules (we documented he
