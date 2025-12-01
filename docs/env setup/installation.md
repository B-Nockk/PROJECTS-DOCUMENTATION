Good catch on both questions!

## Issue 1: Conda Installation Error

The error means your system Python (3.12) is being used instead of Conda's Python. Let's fix this:

```bash
# First, verify you're actually in the conda environment
conda activate backend
which python  # Should show path with "conda" or "envs/backend" in it

# If it shows /usr/bin/python, conda isn't activated properly
# Deactivate and reactivate
conda deactivate
conda activate backend

# Now check Python version
python --version  # Should show the version you specified

# Install packages using conda first (preferred for core packages)
conda install -c conda-forge django=5.0.0 -y

# For packages not available in conda, use pip (which now uses conda's pip)
pip install djangorestframework==3.14.0 \
            psycopg2-binary==2.9.9 \
            django-cors-headers==4.3.0 \
            cryptography==41.0.7 \
            python-decouple==3.8 \
            fastapi==0.109.0 \
            uvicorn[standard]==0.27.0 \
            httpx==0.26.0 \
            pydantic==2.5.0
```

**If that still fails**, it means your conda environment isn't properly isolated. Try this nuclear option:

```bash
# Remove and recreate environment
conda deactivate
conda env remove -n backend -y

# Create with explicit Python version
conda create -n backend python=3.11 pip -y
conda activate backend

# Verify isolation
which pip  # Should show conda path, NOT /usr/bin/pip

# Now install everything
pip install django djangorestframework psycopg2-binary \
            django-cors-headers cryptography python-decouple \
            fastapi uvicorn[standard] httpx pydantic
```

---

## Issue 2: Database Operations - YES, Django ORM Only

**Correct!** Here's how it works:

### Architecture Clarification

```
┌─────────────────────────────────────────┐
│         FastAPI Service (8000)          │
│  ┌─────────────────────────────────┐   │
│  │  Does NOT touch database        │   │
│  │  directly                        │   │
│  └──────────────┬──────────────────┘   │
│                 │ HTTP Request          │
│                 ▼                       │
│  ┌─────────────────────────────────┐   │
│  │  Calls Django REST API           │   │
│  │  to fetch secrets                │   │
│  └─────────────────────────────────┘   │
└─────────────────┬───────────────────────┘
                  │
                  │ GET /api/providers/stripe/secret
                  ▼
┌─────────────────────────────────────────┐
│         Django Service (8001)           │
│  ┌─────────────────────────────────┐   │
│  │  Django ORM (ONLY DB access)    │   │
│  │  WebhookProvider.objects.get()  │   │
│  └──────────────┬──────────────────┘   │
│                 │                       │
│                 ▼                       │
│         PostgreSQL/SQLite               │
└─────────────────────────────────────────┘
```

### Why This Architecture?

1. **Single Source of Truth**: Django ORM manages ALL database operations
2. **No Duplicate Logic**: Don't need to configure DB connection in both services
3. **Security**: FastAPI can't accidentally bypass Django's encryption logic
4. **Django Admin Works**: Admin interface uses same ORM models

### How FastAPI Gets Data

**Option A: Internal API Call (What We'll Use)**

```python
# FastAPI code
import httpx

async def get_provider_secret(provider_name: str) -> str:
    async with httpx.AsyncClient() as client:
        response = await client.get(
            f"http://django-service:8001/api/providers/{provider_name}/secret"
        )
        data = response.json()
        return data['secret']  # Django handles decryption
```

**Option B: Shared Database (NOT recommended for learning)**

```python
# Both services connect to same DB
# FastAPI would import Django models
# More complex, breaks separation of concerns
```

### Complete Data Flow Example

```python
# 1. Webhook arrives at FastAPI
POST /webhooks/stripe
Headers: X-Webhook-Signature: abc123...

# 2. FastAPI asks Django for secret
secret = await get_provider_secret("stripe")
# Behind the scenes: Django does WebhookProvider.objects.get(provider="stripe")
# Django decrypts secret_key field automatically

# 3. FastAPI validates signature
is_valid = verify_signature(payload, signature, secret)

# 4. FastAPI tells Django to log the webhook
await httpx.post("http://django:8001/api/webhooks/log", json={
    "provider": "stripe",
    "payload": payload,
    "status": "success" if is_valid else "failed"
})
# Django does: WebhookLog.objects.create(...)
```

### Django API Endpoints We'll Create

```python
# django_service/webhooks/urls.py

# GET /api/providers/              - List all providers
# GET /api/providers/{name}/secret - Get decrypted secret
# POST /api/webhooks/log           - Create webhook log entry
# GET /api/webhooks/               - List recent webhooks
# POST /api/webhooks/{id}/retry    - Retry failed webhook
```

---

## Updated TDD Section

Let me clarify this in the document:

**Database Access Pattern**:

- ✅ Django ORM: ALL database operations
- ✅ FastAPI: Communicates with Django via REST API
- ❌ FastAPI: NEVER touches database directly

**Benefits**:

- Django Admin works seamlessly
- Encryption logic in one place
- FastAPI stays lightweight (just webhook validation)
- Easy to scale services independently

---

## Next: Once Conda Is Fixed

After you get the packages installed successfully, let's verify with:

```bash
# In conda environment
python -c "import django; print(f'Django {django.get_version()} loaded')"
python -c "import fastapi; print('FastAPI loaded')"
python -c "from cryptography.fernet import Fernet; print('Crypto loaded')"
```

**Once those work, we'll create:**

1. Django models with encryption
2. Django REST API endpoints
3. FastAPI service that calls those endpoints

**Did the conda fix work?** Let me know what `which python` and `which pip` show after activating the environment!
