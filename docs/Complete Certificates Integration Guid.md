# Complete Certificates Integration Guide: Your Questions Answered

## Question 1: How Services Read Certs Outside Their Folders

### The Key Insight

**Python imports ≠ File system paths**

```python
# ❌ This is a Python import (uses PYTHONPATH)
from certs.dev import ca_cert  # Doesn't work - certs isn't a Python package

# ✅ This is file system access (uses OS)
with open('./certs/dev/ca-cert.pem', 'rb') as f:
    ca_cert = f.read()  # Works - reading from file system
```

### How It Works

```bash
# Your project structure
/home/nockk/portfolio/webhook/  ← Your CWD (Current Working Directory)
├── certs/
│   └── dev/
│       └── ca-cert.pem
├── django_service/
│   └── grpc_server.py
└── fastapi_service/
    └── grpc_client.py

# When you run:
$ python django_service/grpc_server.py

# The process runs with CWD = /home/nockk/portfolio/webhook/
# So './certs/dev/ca-cert.pem' resolves to:
#    /home/nockk/portfolio/webhook/certs/dev/ca-cert.pem
```

**Both services can read the same files because:**

1. They run from the same CWD (project root)
2. File paths are relative to CWD, not to the script location
3. The OS handles file access, not Python's import system

### Proof of Concept

```python
# django_service/grpc_server.py
import os
from pathlib import Path

# Where is this script?
script_location = Path(__file__).parent
print(f"Script location: {script_location}")
# Output: /home/nockk/portfolio/webhook/django_service

# Where is the process running from?
cwd = Path.cwd()
print(f"Current directory: {cwd}")
# Output: /home/nockk/portfolio/webhook

# Relative path resolves from CWD, not script location
cert_path = Path('./certs/dev/ca-cert.pem')
absolute = cert_path.resolve()
print(f"Certificate resolves to: {absolute}")
# Output: /home/nockk/portfolio/webhook/certs/dev/ca-cert.pem
```

---

## Question 2: Where to Integrate Certs in Django

### Answer: In `grpc_server.py`, NOT in servicers

**Architecture:**

```
grpc_server.py (Transport Layer)
    ↓ Handles: Security, TLS, Authentication
    ↓ Loads: Certificates
    ↓
    ├─→ OrganizationServicer (Business Logic)
    │       ↓ Handles: CRUD operations
    │       ↓ Doesn't know about: Certificates, network, transport
    │
    └─→ UserServicer (Business Logic)
            ↓ Handles: User operations
            ↓ Doesn't know about: Certificates, network, transport
```

### Why This Separation?

**Transport Layer (grpc_server.py):**

- Loads certificates
- Creates secure channel
- Handles TLS handshake
- Authenticates clients
- Manages network connections

**Business Logic (servicers):**

- Validates requests
- Queries database
- Processes data
- Returns responses
- No knowledge of transport security

### Where Each Piece Goes

```python
# grpc_server.py (YOU LOAD CERTS HERE)
server_credentials = grpc.ssl_server_credentials(
    private_key_certificate_chain_pairs=[(server_key, server_cert)],
    root_certificates=ca_cert,
    require_client_auth=True,
)
server.add_secure_port(f"[::]:{port}", server_credentials)

# organization_servicer.py (NO CERTIFICATES HERE)
class OrganizationServicerImpl(OrganizationServiceServicer):
    def GetOrgById(self, request, context):
        # Just business logic
        org = Organization.objects.get(id=request.id)
        return organization_pb2.Organization(...)
```

---

## Question 3: Django Environment Loading

### Does `manage.py` Need to Be Aware?

**Yes, if you want Django commands to use environment-specific settings.**

### The Problem

```bash
# Without environment loading
$ python manage.py runserver
# Django doesn't know about your .env files
# Uses default settings or system environment variables
```

### The Solution

Update `manage.py` to load environment files (see artifact: django_manage_aware)

```python
# django_service/manage.py
def load_environment():
    raw_env_mode = os.getenv("ENV_MODE") or "dev"  # Default to dev
    env_file = Path(__file__).parent / f".env.{raw_env_mode}"
    if env_file.exists():
        load_dotenv(env_file, override=True)
        print(f"✅ Loaded environment from {env_file.name}")
    return raw_env_mode

def main():
    load_environment()  # Load BEFORE Django starts
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "django_service.config.settings")
    # ... rest of manage.py
```

### Now Django Commands Work with Environments

```bash
# Run server in dev mode
ENV_MODE=dev python manage.py runserver

# Run migrations in staging
ENV_MODE=staging python manage.py migrate

# Collect static files for prod
ENV_MODE=prod python manage.py collectstatic
```

### Using Environment Variables in Django Settings

```python
# django_service/config/settings.py
import os

# These are now loaded from .env files!
DEBUG = os.getenv('DEBUG', 'False') == 'True'
DATABASE_URL = os.getenv('DATABASE_URL', 'sqlite:///db.sqlite3')

# You can also use them for any other settings
ALLOWED_HOSTS = os.getenv('ALLOWED_HOSTS', 'localhost,127.0.0.1').split(',')
```

---

## Complete Setup Steps

### 1. Create Environment Files

```bash
# Django service
cat > django_service/.env.dev << 'EOF'
GRPC_SERVER_PORT=50051
GRPC_SERVER_CERT_PATH=./certs/dev/server-cert.pem
GRPC_SERVER_KEY_PATH=./certs/dev/server-key.pem
GRPC_CA_PATH=./certs/dev/ca-cert.pem
DEBUG=True
EOF

# FastAPI service (already exists)
# fastapi_service/.env.dev already has the config
```

### 2. Update manage.py

Replace your `django_service/manage.py` with the environment-aware version (see artifact).

### 3. Update grpc_server.py

Replace your `django_service/grpc_server.py` with the certificate-enabled version (see artifact).

### 4. Test the Setup

```bash
# Terminal 1: Start Django gRPC Server
cd /home/nockk/portfolio/webhook
ENV_MODE=dev python django_service/grpc_server.py

# Should see:
# ✅ Loaded environment from .env.dev
# 🔧 Environment: dev
# 📡 gRPC Port: 50051
# 🔐 Server Cert: ./certs/dev/server-cert.pem
# ✅ Certificates loaded successfully
# 🚀 Secure gRPC Server Started

# Terminal 2: Start FastAPI
cd /home/nockk/portfolio/webhook
ENV_MODE=dev python -m uvicorn fastapi_service.main:app --reload

# Should see:
# ✅ Loaded environment from .env.dev
# 🔧 Environment: dev
# 🌐 HTTP Server: 0.0.0.0:8000
# 📡 gRPC Server: localhost:50051
# 🔍 Checking gRPC connection...
# ✅ gRPC connection healthy

# Terminal 3: Test the connection
curl http://localhost:8000/health

# Should return:
# {
#   "status": "healthy",
#   "checks": {
#     "http": "ok",
#     "grpc": "ok"
#   }
# }
```

---

## Common Issues and Solutions

### Issue 1: "Certificate file not found"

**Cause:** Running from wrong directory

```bash
# ❌ Wrong - CWD is django_service/
cd django_service
python grpc_server.py
# Error: ./certs/dev/ca-cert.pem not found

# ✅ Correct - CWD is project root
cd /home/nockk/portfolio/webhook
python django_service/grpc_server.py
```

### Issue 2: "Cannot import django_service"

**Cause:** Project root not in PYTHONPATH

```python
# grpc_server.py handles this:
PROJECT_ROOT = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
if PROJECT_ROOT not in sys.path:
    sys.path.insert(0, PROJECT_ROOT)
```

### Issue 3: "Environment file not found"

**Cause:** Missing .env files

```bash
# Check if files exist
ls django_service/.env.dev
ls fastapi_service/.env.dev

# Create if missing
touch django_service/.env.dev
touch fastapi_service/.env.dev
```

### Issue 4: gRPC connection fails

**Debugging steps:**

```bash
# 1. Check if Django gRPC server is running
lsof -i :50051

# 2. Verify certificates exist
ls -lh certs/dev/

# 3. Test certificate validity
make certs-verify

# 4. Check environment variables
ENV_MODE=dev python -c "
from dotenv import load_dotenv
from pathlib import Path
load_dotenv('fastapi_service/.env.dev')
import os
print('GRPC_SERVER_ADDRESS:', os.getenv('GRPC_SERVER_ADDRESS'))
print('GRPC_CERT_PATH:', os.getenv('GRPC_CERT_PATH'))
"
```

---

## Makefile Commands

```bash
# Generate certificates
make certs-dev

# Verify certificates
make certs-verify

# Generate proto files
make proto-all

# Start Django gRPC server
ENV_MODE=dev python django_service/grpc_server.py

# Start FastAPI
ENV_MODE=dev python -m uvicorn fastapi_service.main:app --reload

# Run Django migrations
ENV_MODE=dev python django_service/manage.py migrate

# Run Django HTTP server (separate from gRPC)
ENV_MODE=dev python django_service/manage.py runserver 8001
```

---

## Key Takeaways

1. **File paths work differently from imports**

   - Services read certs from file system (OS operation)
   - Not Python imports (module system)
   - Works because they share the same CWD

2. **Certificates go in `grpc_server.py`**

   - Transport layer handles security
   - Servicers handle business logic
   - Clear separation of concerns

3. **`manage.py` can load environments**

   - Modify it to load .env files
   - Django commands then use environment-specific config
   - Consistent with FastAPI approach

4. **Always run from project root**

   - Ensures relative paths resolve correctly
   - Both services find certificates
   - Import paths work properly

5. **Environment variables make it portable**
   - Same code works in dev/staging/prod
   - Just change ENV_MODE
   - No code changes needed
