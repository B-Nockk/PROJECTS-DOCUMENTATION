# Proto Tools - Quick Reference Card

## 🚀 Most Used Commands

| Command           | What It Does                             |
| ----------------- | ---------------------------------------- |
| `make proto-all`  | Generate ALL protos for BOTH services ⭐ |
| `make proto-list` | Show available .proto files              |
| `make proto-help` | Show all available commands              |

## 📦 Service-Specific

| Command              | Service(s)    |
| -------------------- | ------------- |
| `make proto-django`  | Django only   |
| `make proto-fastapi` | FastAPI only  |
| `make proto-all`     | Both services |

## 🎯 Individual Files (Django)

| Command              | Proto File         |
| -------------------- | ------------------ |
| `make proto-user`    | user.proto         |
| `make proto-org`     | organization.proto |
| `make proto-webhook` | webhook.proto      |

## 🧪 Development Workflow

```bash
# 1. Add new proto
touch django_service/protos/webhook_log.proto

# 2. Check it's detected
make proto-list

# 3. Generate for both services
make proto-all

# 4. Start using in code
```

## 🔧 Direct Script Usage

```bash
# Generate all for both services
python scripts/proto_tools.py generate --all --service both

# Generate specific files
python scripts/proto_tools.py generate --proto user.proto organization.proto

# Generate for FastAPI
python scripts/proto_tools.py generate --proto user.proto --service fastapi

# List available
python scripts/proto_tools.py list
```

## 📁 What Gets Generated

```
user.proto  →  user_pb2.py + user_pb2_grpc.py
```

Generated in:

- **Django**: `django_service/webhooks/grpc_generated/`
- **FastAPI**: `fastapi_service/grpc_generated/`

## 🔥 Hot Tips

1. **Default is smart**: Just use `make proto-all` for everything
2. **Iterate fast**: Use `make proto-org` during development
3. **Always sync**: Run `make proto-all` before committing
4. **After pulling**: Run `make proto-all` to sync generated code

## 🆘 Common Issues

**Proto not found?**

```bash
make proto-list  # Check if file exists
```

**Import errors?**

```bash
make proto-clean
make proto-all
```

**Missing grpc_tools?**

```bash
pip install grpcio-tools
```

## 💡 Remember

- Proto files live in: `django_service/protos/`
- Each service gets its own generated code (no sharing!)
- Always generate for both services before deployment
- Commit generated files to git
