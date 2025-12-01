# Proto Tools Usage Guide

## Quick Start

### Most Common Command (Generate Everything)

```bash
make proto-all
# or
python scripts/proto_tools.py generate --all --service both
```

This generates all `.proto` files for both Django and FastAPI services. Use this:

- After adding new proto files
- During initial setup
- When switching branches
- When proto definitions change

---

## Makefile Commands

### Essential Commands

```bash
# Generate all protos for both services (most common)
make proto-all

# List available proto files
make proto-list

# Show all available commands
make proto-help
```

### Service-Specific Generation

```bash
# Generate for Django only
make proto-django

# Generate for FastAPI only
make proto-fastapi
```

### Individual File Generation (Django only)

```bash
# Generate specific proto for Django
make proto-user
make proto-org
make proto-webhook

# Generate multiple core protos
make proto-core  # Generates user.proto + organization.proto
```

### Cleanup

```bash
# Remove all generated files
make proto-clean
```

---

## Direct Script Usage

### Generate All Files

```bash
# For both services (recommended)
python scripts/proto_tools.py generate --all --service both

# For Django only
python scripts/proto_tools.py generate --all --service django

# For FastAPI only
python scripts/proto_tools.py generate --all --service fastapi
```

### Generate Specific Files

```bash
# Single file (Django - default)
python scripts/proto_tools.py generate --proto user.proto

# Multiple files (Django)
python scripts/proto_tools.py generate --proto user.proto organization.proto

# Single file for FastAPI
python scripts/proto_tools.py generate --proto user.proto --service fastapi

# Multiple files for both services
python scripts/proto_tools.py generate --proto user.proto organization.proto --service both
```

### Custom Output Directory

```bash
# Generate to custom location
python scripts/proto_tools.py generate --proto user.proto --out custom/path/grpc_generated
```

### List Available Files

```bash
python scripts/proto_tools.py list
```

---

## Common Workflows

### 1. Initial Project Setup

```bash
# First time setup
make proto-all

# Verify generation
make proto-list
```

### 2. Adding a New Proto File

```bash
# 1. Create new proto file
touch django_service/protos/webhook_log.proto

# 2. Define your proto
vim django_service/protos/webhook_log.proto

# 3. List to verify it's detected
make proto-list

# 4. Generate for both services
make proto-all
```

### 3. Modifying Existing Proto

```bash
# 1. Edit the proto file
vim django_service/protos/organization.proto

# 2. Regenerate for both services
make proto-all

# Or just for the service you're testing
make proto-django
```

### 4. Working on Single Proto (Development)

```bash
# Faster iteration - only regenerate what you need
make proto-org

# Test in Django
make run-grpc

# When satisfied, generate for FastAPI too
make proto-fastapi
```

### 5. Cleaning Up

```bash
# Remove all generated files
make proto-clean

# Regenerate everything
make proto-all
```

---

## Directory Structure

```
webhook/
├── django_service/
│   ├── protos/                          # Source .proto files
│   │   ├── user.proto
│   │   ├── organization.proto
│   │   └── webhook.proto
│   └── webhooks/
│       └── grpc_generated/              # Generated Django code
│           ├── __init__.py
│           ├── user_pb2.py
│           ├── user_pb2_grpc.py
│           ├── organization_pb2.py
│           └── organization_pb2_grpc.py
│
├── fastapi_service/
│   └── grpc_generated/                  # Generated FastAPI code
│       ├── __init__.py
│       ├── user_pb2.py
│       ├── user_pb2_grpc.py
│       ├── organization_pb2.py
│       └── organization_pb2_grpc.py
│
└── scripts/
    └── proto_tools.py                   # Generation script
```

---

## What Gets Generated?

For each `.proto` file, you get two Python files:

### Example: `user.proto`

**Generated files:**

- `user_pb2.py` - Message definitions (data structures)
- `user_pb2_grpc.py` - Service stubs and servicers (RPC methods)

**Django service gets:**

```python
# In django_service/webhooks/grpc_generated/
user_pb2.py           # Messages: User, UserIdRequest, etc.
user_pb2_grpc.py      # Service: UserServiceServicer (server implementation)
```

**FastAPI service gets:**

```python
# In fastapi_service/grpc_generated/
user_pb2.py           # Same messages: User, UserIdRequest, etc.
user_pb2_grpc.py      # Service: UserServiceStub (client to call Django)
```

---

## Troubleshooting

### Error: "Proto file not found"

```bash
# Check available files
make proto-list

# Verify file exists
ls django_service/protos/
```

### Error: "grpc_tools.protoc not found"

```bash
# Install required package
pip install grpcio-tools
```

### Generated Files Not Importing

```bash
# Clean and regenerate
make proto-clean
make proto-all

# Verify __init__.py exists in output directories
ls django_service/webhooks/grpc_generated/__init__.py
ls fastapi_service/grpc_generated/__init__.py
```

### Want to See What Will Be Generated

```bash
# List available protos
make proto-list

# Output:
# 📋 Available proto files in django_service/protos:
#    - organization.proto
#    - user.proto
#    - webhook.proto
```

---

## Best Practices

### 1. Always Generate for Both Services

Unless you're actively developing and iterating, always use:

```bash
make proto-all
```

### 2. Commit Generated Files

Generated files should be committed to version control so that:

- CI/CD doesn't need proto compilation
- Teammates get consistent code
- Deployments are reproducible

### 3. Regenerate After Pulling Changes

```bash
git pull
make proto-all  # Ensure generated code matches proto definitions
```

### 4. Use Descriptive Proto Names

```
✅ Good:
  - user.proto
  - organization.proto
  - webhook_log.proto
  - audit_log.proto

❌ Avoid:
  - proto1.proto
  - test.proto
  - temp.proto
```

### 5. Keep Protos in Version Control

```bash
# Track proto source files
git add django_service/protos/*.proto

# Track generated files
git add django_service/webhooks/grpc_generated/
git add fastapi_service/grpc_generated/
```

---

## Advanced Usage

### Generate with Custom Proto Directory

```bash
python scripts/proto_tools.py generate \
    --all \
    --proto-dir custom/protos \
    --service both
```

### Generate to Custom Output

```bash
python scripts/proto_tools.py generate \
    --proto user.proto \
    --out build/generated/grpc
```

### Scripting / CI/CD

```bash
#!/bin/bash
# In CI/CD pipeline

# Generate all protos
python scripts/proto_tools.py generate --all --service both

# Verify generation succeeded
if [ $? -ne 0 ]; then
    echo "Proto generation failed"
    exit 1
fi

echo "Proto generation successful"
```

---

## Summary

**For 99% of use cases:**

```bash
make proto-all
```

**For development iteration:**

```bash
make proto-list        # See what's available
make proto-org         # Generate just what you're working on
make proto-django      # Generate all for Django when ready
make proto-all         # Generate everything before committing
```

The tool is designed to be smart by default but flexible when you need it!
