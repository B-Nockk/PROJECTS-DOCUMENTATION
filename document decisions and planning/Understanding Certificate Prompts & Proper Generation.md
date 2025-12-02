# Understanding Certificate Prompts & Proper Generation

## What Those Prompts Mean

When OpenSSL asks for information, it's building the certificate's "Distinguished Name" (DN). This identifies who/what the certificate belongs to.

### Common Prompts Explained

| Prompt                       | What It Means                                      | Does It Limit Access?       |
| ---------------------------- | -------------------------------------------------- | --------------------------- |
| **Country Name (C)**         | Two-letter country code (US, GB, NG, etc.)         | ❌ No - just identification |
| **State/Province (ST)**      | State or province name                             | ❌ No - just identification |
| **City/Locality (L)**        | City name                                          | ❌ No - just identification |
| **Organization (O)**         | Company/organization name                          | ❌ No - just identification |
| **Organizational Unit (OU)** | Department/division name                           | ❌ No - just identification |
| **Common Name (CN)**         | ⚠️ **IMPORTANT** - Server hostname or service name | ✅ **YES** - Must match!    |
| **Email Address**            | Contact email                                      | ❌ No - just identification |

### The CRITICAL One: Common Name (CN)

**Common Name is the ONLY field that affects security:**

For **Server Certificates:**

```bash
Common Name (CN): localhost              # For local dev
Common Name (CN): django-service         # For docker-compose
Common Name (CN): grpc.yourcompany.com   # For production
```

For **Client Certificates:**

```bash
Common Name (CN): fastapi-client         # Service identifier
Common Name (CN): webhook-api            # Or your service name
```

**Why it matters:**

- Client verifies the server's CN matches the address it's connecting to
- If you connect to `localhost:50051` but cert says `example.com`, it fails
- This prevents man-in-the-middle attacks

### Geographic Fields Don't Limit Access

**Common misconception:**

```
❌ "If I put City: Lagos, only Lagos can connect"  - FALSE
❌ "If I put Country: NG, only Nigerian IPs work"  - FALSE
```

**Reality:**

```
✅ These fields are just labels for identification
✅ They appear in cert info but don't restrict anything
✅ You can leave them blank for dev certificates
```

## Example: Reading a Certificate

```bash
openssl x509 -in server-cert.pem -text -noout
```

```
Certificate:
    Subject: C=NG, ST=Lagos, L=Lagos, O=WebhookProcessor, CN=localhost
            └── This is just metadata, like a label
                                                          └── THIS is what matters!

    Subject Alternative Name:
        DNS:localhost, DNS:django-service, IP:127.0.0.1
        └── Even better than CN - allows multiple names
```

---

## Proper Certificate Generation Script

Here's a complete script that asks the right questions and generates proper certificates:

### Script: `scripts/generate_certs.sh`

```bash
#!/bin/bash
# Certificate generation script for gRPC mTLS
# Usage: ./scripts/generate_certs.sh [dev|staging|prod]

set -e  # Exit on error

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Determine environment
ENV=${1:-dev}
CERT_DIR="certs/${ENV}"

echo -e "${GREEN}🔐 Certificate Generation Script${NC}"
echo -e "${YELLOW}Environment: ${ENV}${NC}"
echo ""

# Create directory
mkdir -p "${CERT_DIR}"
cd "${CERT_DIR}"

# Get configuration based on environment
case $ENV in
    dev)
        SERVER_CN="localhost"
        SERVER_ALT_NAMES="DNS:localhost,DNS:django-service,IP:127.0.0.1"
        CLIENT_CN="fastapi-client"
        DAYS=365
        ;;
    staging)
        echo "Enter server hostname (e.g., staging-grpc.yourcompany.com):"
        read SERVER_CN
        SERVER_ALT_NAMES="DNS:${SERVER_CN},DNS:django-service"
        CLIENT_CN="fastapi-client-staging"
        DAYS=365
        ;;
    prod)
        echo "Enter server hostname (e.g., grpc.yourcompany.com):"
        read SERVER_CN
        SERVER_ALT_NAMES="DNS:${SERVER_CN},DNS:django-service"
        CLIENT_CN="fastapi-client-prod"
        DAYS=730
        ;;
    *)
        echo -e "${RED}Invalid environment. Use: dev, staging, or prod${NC}"
        exit 1
        ;;
esac

echo -e "${YELLOW}📝 Certificate Configuration:${NC}"
echo "   Server CN: ${SERVER_CN}"
echo "   Server Alt Names: ${SERVER_ALT_NAMES}"
echo "   Client CN: ${CLIENT_CN}"
echo "   Valid for: ${DAYS} days"
echo ""

# Ask for organization details (optional for dev)
if [ "$ENV" = "dev" ]; then
    echo -e "${YELLOW}For development, we'll use default values.${NC}"
    COUNTRY="NG"
    STATE="Lagos"
    CITY="Lagos"
    ORG="WebhookProcessor-Dev"
    OU="Engineering"
else
    echo "Enter organization details:"
    echo -n "Country (2 letters, e.g., NG, US): "
    read COUNTRY
    echo -n "State/Province: "
    read STATE
    echo -n "City: "
    read CITY
    echo -n "Organization: "
    read ORG
    echo -n "Organizational Unit: "
    read OU
fi

echo ""
echo -e "${GREEN}Generating certificates...${NC}"

# -------------------------------------------------------------------
# 1. Generate Certificate Authority (CA)
# -------------------------------------------------------------------
echo "1️⃣  Generating CA..."

openssl genrsa -out ca-key.pem 4096

openssl req -new -x509 -days $DAYS -key ca-key.pem -out ca-cert.pem \
    -subj "/C=${COUNTRY}/ST=${STATE}/L=${CITY}/O=${ORG}/OU=${OU}/CN=${ORG} CA"

echo -e "${GREEN}   ✅ CA generated${NC}"

# -------------------------------------------------------------------
# 2. Generate Server Certificate (Django gRPC Server)
# -------------------------------------------------------------------
echo "2️⃣  Generating Server Certificate..."

openssl genrsa -out server-key.pem 4096

openssl req -new -key server-key.pem -out server.csr \
    -subj "/C=${COUNTRY}/ST=${STATE}/L=${CITY}/O=${ORG}/OU=${OU}/CN=${SERVER_CN}"

# Create extension file for Subject Alternative Names
cat > server-ext.cnf <<EOF
subjectAltName = ${SERVER_ALT_NAMES}
extendedKeyUsage = serverAuth
EOF

openssl x509 -req -days $DAYS -in server.csr \
    -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial \
    -out server-cert.pem \
    -extfile server-ext.cnf

echo -e "${GREEN}   ✅ Server certificate generated${NC}"

# -------------------------------------------------------------------
# 3. Generate Client Certificate (FastAPI Client)
# -------------------------------------------------------------------
echo "3️⃣  Generating Client Certificate..."

openssl genrsa -out client-key.pem 4096

openssl req -new -key client-key.pem -out client.csr \
    -subj "/C=${COUNTRY}/ST=${STATE}/L=${CITY}/O=${ORG}/OU=${OU}/CN=${CLIENT_CN}"

# Create extension file for client auth
cat > client-ext.cnf <<EOF
extendedKeyUsage = clientAuth
EOF

openssl x509 -req -days $DAYS -in client.csr \
    -CA ca-cert.pem -CAkey ca-key.pem -CAcreateserial \
    -out client-cert.pem \
    -extfile client-ext.cnf

echo -e "${GREEN}   ✅ Client certificate generated${NC}"

# -------------------------------------------------------------------
# 4. Set Proper Permissions
# -------------------------------------------------------------------
echo "4️⃣  Setting permissions..."

chmod 600 *-key.pem ca-key.pem
chmod 644 *-cert.pem ca-cert.pem

echo -e "${GREEN}   ✅ Permissions set${NC}"

# -------------------------------------------------------------------
# 5. Verify Certificates
# -------------------------------------------------------------------
echo ""
echo -e "${GREEN}🔍 Verifying Certificates...${NC}"

echo "CA Certificate:"
openssl x509 -in ca-cert.pem -noout -subject -issuer -dates

echo ""
echo "Server Certificate:"
openssl x509 -in server-cert.pem -noout -subject -issuer -dates
openssl x509 -in server-cert.pem -noout -text | grep -A1 "Subject Alternative Name"

echo ""
echo "Client Certificate:"
openssl x509 -in client-cert.pem -noout -subject -issuer -dates

# Verify cert chains
echo ""
echo "Verifying certificate chains..."
openssl verify -CAfile ca-cert.pem server-cert.pem
openssl verify -CAfile ca-cert.pem client-cert.pem

# -------------------------------------------------------------------
# 6. Summary
# -------------------------------------------------------------------
echo ""
echo -e "${GREEN}✅ Certificate Generation Complete!${NC}"
echo ""
echo "Generated files in ${CERT_DIR}:"
echo "  📜 ca-cert.pem       - Certificate Authority (shared)"
echo "  🔑 ca-key.pem        - CA private key (KEEP SECRET!)"
echo "  📜 server-cert.pem   - Django server certificate"
echo "  🔑 server-key.pem    - Django server private key"
echo "  📜 client-cert.pem   - FastAPI client certificate"
echo "  🔑 client-key.pem    - FastAPI client private key"
echo ""
echo -e "${YELLOW}Next Steps:${NC}"
echo "  1. Update .env files with certificate paths"
echo "  2. Never commit *-key.pem files to git"
echo "  3. For production, use a secrets management service"
echo ""
echo -e "${YELLOW}Django gRPC Server needs:${NC}"
echo "  GRPC_SERVER_CERT_PATH=./certs/${ENV}/server-cert.pem"
echo "  GRPC_SERVER_KEY_PATH=./certs/${ENV}/server-key.pem"
echo "  GRPC_CA_PATH=./certs/${ENV}/ca-cert.pem"
echo ""
echo -e "${YELLOW}FastAPI Client needs:${NC}"
echo "  GRPC_CERT_PATH=./certs/${ENV}/client-cert.pem"
echo "  GRPC_KEY_PATH=./certs/${ENV}/client-key.pem"
echo "  GRPC_CA_PATH=./certs/${ENV}/ca-cert.pem"
echo ""

# Clean up temporary files
rm -f server.csr client.csr server-ext.cnf client-ext.cnf

cd - > /dev/null
```

### Make it executable:

```bash
chmod +x scripts/generate_certs.sh
```

### Usage:

```bash
# Development (uses defaults, no prompts)
./scripts/generate_certs.sh dev

# Staging (asks for hostname and details)
./scripts/generate_certs.sh staging

# Production (asks for hostname and details)
./scripts/generate_certs.sh prod
```

---

## Template Answers for Certificate Prompts

### Development Environment

```bash
Country Name (C): NG
State/Province (ST): Lagos
City (L): Lagos
Organization (O): WebhookProcessor-Dev
Organizational Unit (OU): Engineering
Common Name (CN): localhost              # ⚠️ CRITICAL - must match!
Email Address: [leave blank or dev@localhost]
```

### Staging/Production Environment

```bash
Country Name (C): NG
State/Province (ST): Lagos
City (L): Lagos
Organization (O): YourCompanyName
Organizational Unit (OU): Platform Engineering
Common Name (CN): grpc.yourcompany.com   # ⚠️ CRITICAL - your actual hostname!
Email Address: platform@yourcompany.com
```

---

## What Matters vs What Doesn't

### ✅ Critical for Security:

1. **Common Name (CN)** - Must match the hostname you're connecting to
2. **Subject Alternative Names (SAN)** - Even better than CN, allows multiple names
3. **Key size** - 4096 bits is good, 2048 minimum
4. **Expiry date** - Don't set too long (1 year is good for dev, shorter for prod)

### ❌ Doesn't Affect Security:

1. Country, State, City - Just labels
2. Organization, OU - Just labels
3. Email - Just for contact info
4. Serial number - Auto-generated

---

## Quick Fixes for Your Current Certs

If you want to quickly regenerate with proper settings:

```bash
# 1. Backup current certs
mkdir -p certs/backup
mv *.pem *.csr *.srl certs/backup/ 2>/dev/null || true

# 2. Generate new ones
./scripts/generate_certs.sh dev

# 3. Test
make proto-all
make run-grpc
# In another terminal
make run-fastapi
```

---

## Summary

**The Short Answer:**

- Geographic info (Country, City, etc.) = Just labels, leave blank if you want
- Common Name = Critical, must match your hostname
- For local dev: CN should be "localhost"
- Keep private keys secret, never commit them

**For your current setup:**

```bash
# Just regenerate with the script
./scripts/generate_certs.sh dev
```

It will use proper defaults for dev and you'll be all set!
