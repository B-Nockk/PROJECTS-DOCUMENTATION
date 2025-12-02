# Certificate Organization for gRPC Services

## Recommended Directory Structure

```
webhook/
├── certs/
│   ├── dev/                          # Development certificates
│   │   ├── ca-cert.pem              # Certificate Authority (shared)
│   │   ├── ca-key.pem               # CA private key (KEEP SECRET)
│   │   ├── server-cert.pem          # Django gRPC server certificate
│   │   ├── server-key.pem           # Django server private key
│   │   ├── client-cert.pem          # FastAPI client certificate
│   │   └── client-key.pem           # FastAPI client private key
│   │
│   ├── staging/                      # Staging environment (future)
│   │   └── [same structure]
│   │
│   └── prod/                         # Production environment (future)
│       └── [same structure]
│
├── django_service/
│   └── (NO certificates here - use environment variables)
│
└── fastapi_service/
    └── (NO certificates here - use environment variables)
```

## Why This Structure?

### ✅ DO: Keep certificates in a shared `certs/` directory

**Benefits:**

- Central management of all certificates
- Easy to rotate/update certificates
- Clear separation by environment
- Services reference via environment variables
- Easy to exclude from git (security)

### ❌ DON'T: Copy certificates into service directories

**Problems:**

- Duplication (hard to update)
- Security risk (easier to accidentally commit)
- Unclear which cert is "source of truth"
- Harder to manage across environments

## Certificate Roles Explained

### 1. Certificate Authority (CA)

```
ca-cert.pem  - Public certificate of your CA
ca-key.pem   - Private key (KEEP SECRET!)
```

**What it does:**

- Acts as the "trust anchor" for your system
- Signs all other certificates
- Both services trust this CA to verify each other

**Think of it as:** Your own mini "Verisign" or "Let's Encrypt"

### 2. Server Certificates (Django gRPC Server)

```
server-cert.pem  - Django's identity certificate
server-key.pem   - Django's private key
```

**What it does:**

- Proves "I am the real Django gRPC server"
- FastAPI verifies this certificate against the CA
- Encrypts data in transit

### 3. Client Certificates (FastAPI Service)

```
client-cert.pem  - FastAPI's identity certificate
client-key.pem   - FastAPI's private key
```

**What it does:**

- Proves "I am an authorized FastAPI client"
- Django verifies this certificate against the CA
- Ensures only authorized clients can connect

## Who Needs What?

### Django gRPC Server Needs:

```bash
GRPC_SERVER_CERT=/path/to/certs/dev/server-cert.pem
GRPC_SERVER_KEY=/path/to/certs/dev/server-key.pem
GRPC_CA_CERT=/path/to/certs/dev/ca-cert.pem
```

### FastAPI Client Needs:

```bash
GRPC_CLIENT_CERT=/path/to/certs/dev/client-cert.pem
GRPC_CLIENT_KEY=/path/to/certs/dev/client-key.pem
GRPC_CA_CERT=/path/to/certs/dev/ca-cert.pem
```

### Both Services Share:

- `ca-cert.pem` - The trust anchor

## Security Notes

### What to Commit to Git:

```gitignore
# .gitignore

# NEVER commit private keys
certs/**/*-key.pem
certs/**/ca-key.pem

# For dev, you CAN commit certificates (public keys)
# They're not secret, just identity
# But best practice is to not commit any certs

# For staging/prod - NEVER commit anything
certs/staging/
certs/prod/

# Alternative: Don't commit any certs
certs/
```

### What to Keep Secret:

- ❌ `ca-key.pem` - Anyone with this can create trusted certificates!
- ❌ `server-key.pem` - Private keys must stay private
- ❌ `client-key.pem` - Private keys must stay private

### What's Safe to Share:

- ✅ `ca-cert.pem` - Public certificate (but still best practice to not commit)
- ✅ `server-cert.pem` - Public certificate
- ✅ `client-cert.pem` - Public certificate

**For dev environment:** You can commit certs but not keys
**For staging/prod:** Use secrets management (AWS Secrets Manager, HashiCorp Vault, etc.)

## File Permissions (Important!)

```bash
# Set correct permissions
cd certs/dev

# Private keys should be readable only by owner
chmod 600 *-key.pem
chmod 600 ca-key.pem

# Certificates can be more permissive
chmod 644 *-cert.pem
chmod 644 ca-cert.pem

# Verify
ls -la
# Should see:
# -rw------- 1 user user 3243 Dec 01 10:00 ca-key.pem
# -rw-r--r-- 1 user user 1234 Dec 01 10:00 ca-cert.pem
# -rw------- 1 user user 3243 Dec 01 10:00 server-key.pem
# -rw-r--r-- 1 user user 1234 Dec 01 10:00 server-cert.pem
# -rw------- 1 user user 3243 Dec 01 10:00 client-key.pem
# -rw-r--r-- 1 user user 1234 Dec 01 10:00 client-cert.pem
```

## Migration Steps

### 1. Create Certificate Directory

```bash
# From project root
mkdir -p certs/dev
mkdir -p certs/staging
mkdir -p certs/prod
```

### 2. Move Your Certificates

```bash
# Move existing certs
mv ca-cert.pem certs/dev/
mv ca-key.pem certs/dev/
mv ca-cert.srl certs/dev/
mv server-cert.pem certs/dev/
mv server-key.pem certs/dev/
mv server.csr certs/dev/
mv client-cert.pem certs/dev/
mv client-key.pem certs/dev/
mv client.csr certs/dev/
```

### 3. Set Permissions

```bash
chmod 600 certs/dev/*-key.pem
chmod 644 certs/dev/*-cert.pem
```

### 4. Update Environment Files

**fastapi_service/.env.dev:**

```bash
GRPC_SERVER_ADDRESS=localhost:50051
GRPC_CERT_PATH=./certs/dev/client-cert.pem
GRPC_KEY_PATH=./certs/dev/client-key.pem
GRPC_CA_PATH=./certs/dev/ca-cert.pem
```

**django_service/.env.dev** (you'll need to create this):

```bash
GRPC_SERVER_PORT=50051
GRPC_SERVER_CERT_PATH=./certs/dev/server-cert.pem
GRPC_SERVER_KEY_PATH=./certs/dev/server-key.pem
GRPC_CA_PATH=./certs/dev/ca-cert.pem
```

### 5. Update .gitignore

```bash
# Add to .gitignore
echo "" >> .gitignore
echo "# Certificates" >> .gitignore
echo "certs/**/*-key.pem" >> .gitignore
echo "certs/**/ca-key.pem" >> .gitignore
echo "certs/staging/" >> .gitignore
echo "certs/prod/" >> .gitignore
```

## Docker Considerations

When running in Docker, mount the certs directory:

```yaml
# docker-compose.yml
services:
  django-service:
    volumes:
      - ./certs/dev:/app/certs/dev:ro # Read-only mount
    environment:
      - GRPC_SERVER_CERT_PATH=/app/certs/dev/server-cert.pem
      - GRPC_SERVER_KEY_PATH=/app/certs/dev/server-key.pem
      - GRPC_CA_PATH=/app/certs/dev/ca-cert.pem

  fastapi-service:
    volumes:
      - ./certs/dev:/app/certs/dev:ro # Read-only mount
    environment:
      - GRPC_CERT_PATH=/app/certs/dev/client-cert.pem
      - GRPC_KEY_PATH=/app/certs/dev/client-key.pem
      - GRPC_CA_PATH=/app/certs/dev/ca-cert.pem
```

## Summary

**Best Practice Structure:**

```
certs/
  └── dev/
      ├── ca-cert.pem       # Shared by both services
      ├── ca-key.pem        # KEEP SECRET, used to sign certs
      ├── server-cert.pem   # Django identity
      ├── server-key.pem    # Django private key
      ├── client-cert.pem   # FastAPI identity
      └── client-key.pem    # FastAPI private key
```

**Reference via environment variables:**

- Never hardcode paths in code
- Easy to change per environment
- Services stay portable

**Security:**

- Private keys (600 permissions)
- Never commit private keys to git
- Use secrets management for staging/prod
