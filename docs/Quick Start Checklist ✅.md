# Quick Start Checklist ✅

Complete these steps in order to get your secure gRPC communication working.

## Prerequisites

- [ ] Python 3.11 installed
- [ ] Virtual environment activated
- [ ] All dependencies installed (`pip install -r requirements.txt`)
- [ ] Currently in project root: `/home/nockk/portfolio/webhook/`

## Step 1: Environment Files

```bash
# Create Django environment file
cat > django_service/.env.dev << 'EOF'
GRPC_SERVER_PORT=50051
GRPC_SERVER_CERT_PATH=./certs/dev/server-cert.pem
GRPC_SERVER_KEY_PATH=./certs/dev/server-key.pem
GRPC_CA_PATH=./certs/dev/ca-cert.pem
DEBUG=True
DATABASE_URL=sqlite:///db.sqlite3
EOF

# Verify FastAPI environment file exists
cat fastapi_service/.env.dev
```

**Checklist:**

- [ ] `django_service/.env.dev` exists
- [ ] `fastapi_service/.env.dev` exists
- [ ] Both files have certificate paths
- [ ] Ports match (Django: 50051, FastAPI: 8000)

## Step 2: Certificates

```bash
# Generate development certificates
chmod +x scripts/generate_certs.sh
make certs-dev

# Or manually:
./scripts/generate_certs.sh dev

# Verify they were created
ls -lh certs/dev/
```

**Expected files in `certs/dev/`:**

- [ ] `ca-cert.pem` (CA public certificate)
- [ ] `ca-key.pem` (CA private key - 600 permissions)
- [ ] `server-cert.pem` (Django server certificate)
- [ ] `server-key.pem` (Django server key - 600 permissions)
- [ ] `client-cert.pem` (FastAPI client certificate)
- [ ] `client-key.pem` (FastAPI client key - 600 permissions)

```bash
# Verify certificates
make certs-verify
```

- [ ] All certificates verified successfully
- [ ] No "file not found" errors

## Step 3: Proto Code Generation

```bash
# Generate gRPC code for both services
make proto-all

# Verify generated files
ls django_service/webhooks/grpc_generated/
ls fastapi_service/grpc_generated/
```

**Expected generated files:**

- [ ] `organization_pb2.py` in both services
- [ ] `organization_pb2_grpc.py` in both services
- [ ] `user_pb2.py` in both services
- [ ] `user_pb2_grpc.py` in both services

## Step 4: Update Django Files

### Update `manage.py`

Replace `django_service/manage.py` with the environment-aware version:

```python
#!/usr/bin/env python
import os, sys
from pathlib import Path
from dotenv import load_dotenv
from enum import Enum

class EnvMode(str, Enum):
    dev = "dev"
    staging = "staging"
    prod = "prod"

def load_environment():
    raw_env_mode = os.getenv("ENV_MODE", "dev")
    os.environ.setdefault("ENV_MODE", raw_env_mode)

    try:
        env_mode = EnvMode(raw_env_mode)
    except ValueError:
        print(f"❌ Invalid ENV_MODE '{raw_env_mode}'")
        sys.exit(1)

    manage_dir = Path(__file__).parent
    env_file = manage_dir / f".env.{env_mode.value}"

    if env_file.exists():
        load_dotenv(env_file, override=True)
        print(f"✅ Loaded environment from {env_file.name}")

    return env_mode

def main():
    load_environment()
    os.environ.setdefault("DJANGO_SETTINGS_MODULE", "django_service.config.settings")

    try:
        from django.core.management import execute_from_command_line
    except ImportError as exc:
        raise ImportError("Couldn't import Django") from exc

    execute_from_command_line(sys.argv)

if __name__ == "__main__":
    main()
```

- [ ] `manage.py` updated with environment loading

### Update `grpc_server.py`

Replace `django_service/grpc_server.py` with the certificate-enabled version (see artifacts).

- [ ] `grpc_server.py` updated with certificate loading
- [ ] Environment loading added
- [ ] Secure channel creation configured

## Step 5: Test Django gRPC Server

```bash
# Terminal 1: Start Django gRPC server
cd /home/nockk/portfolio/webhook
ENV_MODE=dev python django_service/grpc_server.py
```

**Expected output:**

```
✅ Loaded environment from .env.dev
🔧 Environment: dev
📡 gRPC Port: 50051
🔐 Server Cert: ./certs/dev/server-cert.pem
🔐 CA Cert: ./certs/dev/ca-cert.pem
✅ Certificates loaded successfully
   Server cert: server-cert.pem
   CA cert: ca-cert.pem
   Client authentication: Required (mTLS enabled)
=============================================================
🚀 Secure gRPC Server Started
=============================================================
   Port: 50051
   Environment: dev
   Security: mTLS (mutual TLS authentication)

📡 Available services:
   - webhook_processor.OrganizationService
   - webhook_processor.UserService

🔐 Security Configuration:
   - Client certificates required: Yes
   - Trusted CA: ca-cert.pem

🛑 Press Ctrl+C to stop
=============================================================
```

**Checklist:**

- [ ] Server starts without errors
- [ ] Certificates loaded successfully
- [ ] Port 50051 listening
- [ ] mTLS enabled

## Step 6: Test FastAPI Service

```bash
# Terminal 2: Start FastAPI
cd /home/nockk/portfolio/webhook
ENV_MODE=dev python -m uvicorn fastapi_service.main:app --reload --host 0.0.0.0 --port 8000
```

**Expected output:**

```
✅ Loaded environment from .env.dev
🔧 Environment: dev
🌐 HTTP Server: 0.0.0.0:8000
📡 gRPC Server: localhost:50051
INFO:     Will watch for changes in these directories: ['/home/nockk/portfolio/webhook']
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
🚀 Starting up...
🔍 Checking gRPC connection...
✅ Established secure gRPC channel to localhost:50051
✅ gRPC connection healthy
INFO:     Application startup complete.
```

**Checklist:**

- [ ] FastAPI starts without errors
- [ ] Environment loaded from .env.dev
- [ ] gRPC connection established
- [ ] gRPC health check passes

## Step 7: Verify Communication

```bash
# Terminal 3: Test endpoints
# Health check
curl http://localhost:8000/health

# Expected response:
# {
#   "status": "healthy",
#   "checks": {
#     "http": "ok",
#     "grpc": "ok"
#   }
# }

# Root endpoint
curl http://localhost:8000/

# Expected response:
# {
#   "message": "Webhook Processor API",
#   "status": "online",
#   "environment": "dev"
# }
```

**Checklist:**

- [ ] `/health` returns `"status": "healthy"`
- [ ] gRPC check is `"ok"`
- [ ] No connection errors

## Step 8: Test gRPC RPC Call (Optional)

If you've implemented the organization router:

```bash
# Create an organization via FastAPI (which calls Django via gRPC)
curl -X POST http://localhost:8000/organizations/ \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test Org",
    "slug": "test-org",
    "plan": "free"
  }'

# Get organization
curl http://localhost:8000/organizations/slug/test-org
```

**Checklist:**

- [ ] POST creates organization successfully
- [ ] GET retrieves organization
- [ ] Data flows: FastAPI → gRPC → Django → Database

## Troubleshooting

### If Django gRPC server fails to start:

```bash
# Check if port is already in use
lsof -i :50051

# Verify certificates exist
ls -lh certs/dev/

# Check environment file
cat django_service/.env.dev

# Verify you're in project root
pwd  # Should be: /home/nockk/portfolio/webhook
```

### If FastAPI can't connect to gRPC:

```bash
# Verify Django gRPC is running
lsof -i :50051

# Check FastAPI environment
cat fastapi_service/.env.dev

# Verify address matches
# FastAPI: GRPC_SERVER_ADDRESS=localhost:50051
# Django:  GRPC_SERVER_PORT=50051

# Test certificate files
ls -lh certs/dev/client-cert.pem
ls -lh certs/dev/client-key.pem
ls -lh certs/dev/ca-cert.pem
```

### If certificates are invalid:

```bash
# Regenerate certificates
make certs-clean
make certs-dev

# Verify
make certs-verify
```

## Success Criteria

You've successfully completed setup when:

- [ ] ✅ Both services start without errors
- [ ] ✅ Certificates load successfully
- [ ] ✅ gRPC connection established with mTLS
- [ ] ✅ Health check shows gRPC as healthy
- [ ] ✅ FastAPI can make RPC calls to Django
- [ ] ✅ Data flows between services

## Next Steps

Now that your secure gRPC communication is working:

1. **Implement more RPC endpoints** in Django servicers
2. **Add FastAPI routers** that use gRPC client
3. **Test error handling** (what happens if Django is down?)
4. **Add logging** for gRPC calls
5. **Implement authentication** on FastAPI endpoints
6. **Create staging environment** with separate certificates

## Quick Reference

```bash
# Start Django gRPC server
ENV_MODE=dev python django_service/grpc_server.py

# Start FastAPI
ENV_MODE=dev python -m uvicorn fastapi_service.main:app --reload

# Generate certificates
make certs-dev

# Generate proto code
make proto-all

# Verify certificates
make certs-verify

# Test health
curl http://localhost:8000/health
```

---

**You're all set! 🚀**

Your services are now communicating securely via gRPC with mutual TLS authentication!
