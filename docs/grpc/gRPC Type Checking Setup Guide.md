# gRPC Type Checking Setup Guide

## Overview of Changes

### Key Improvements

1. **Fixed type checking in proto_tools.py** - Uses `pip show` instead of direct import
2. **Modern stub generation** - Uses `--pyi_out` flag (grpcio-tools 1.76+)
3. **Proper Pylance configuration** - Separate from mypy
4. **Clear import fixing** - Automatic relative imports

## Setup Steps

### 1. Install Updated Dependencies

Your conda environment already has the right versions:

- `grpcio==1.76.0` ✅
- `grpcio-tools==1.76.0` ✅
- `mypy-protobuf==3.7.0` ✅

### 2. File Structure

Place the new files:

```
webhook/
├── pyrightconfig.json          # NEW - Pylance configuration
├── .vscode/
│   └── settings.json           # UPDATED
├── pyproject.toml              # UPDATED
└── scripts/
    └── proto_tools.py          # UPDATED
```

### 3. Regenerate Proto Stubs

```bash
# Clean old generated files
make proto-clean

# Generate with type stubs
make proto-stubs-all
```

This will create:

```
grpc_generated/
├── __init__.py
├── common_pb2.py         # Runtime code
├── common_pb2.pyi        # Type stubs ✨
├── user_pb2.py
├── user_pb2.pyi          # Type stubs ✨
├── user_pb2_grpc.py
├── user_pb2_grpc.pyi     # Type stubs ✨
└── ...
```

### 4. Reload VSCode

```
Cmd/Ctrl + Shift + P → "Developer: Reload Window"
```

## About Async gRPC

### What are the advantages?

**Traditional Sync gRPC:**

```python
def CreateUser(self, request, context):
    # Blocks the thread during I/O
    user = database.create_user(request.name)
    return UserResponse(id=user.id)
```

**Async gRPC:**

```python
async def CreateUser(self, request, context):
    # Non-blocking - handles thousands of concurrent requests
    user = await database.create_user(request.name)
    return UserResponse(id=user.id)
```

### Benefits of Async gRPC:

1. **Higher Concurrency** - Handle 1000+ simultaneous requests with same resources
2. **Better Resource Utilization** - Threads don't block during I/O
3. **Modern Python** - Integrates with asyncio ecosystem (FastAPI, aiohttp, etc.)
4. **Database Integration** - Works with async ORMs (asyncpg, Tortoise ORM, etc.)

### When to Use Async:

✅ **Use Async When:**

- High concurrent request volume (>100 simultaneous)
- Lots of I/O operations (database, external APIs)
- Using async frameworks (FastAPI)
- Need to maximize server throughput

❌ **Stick with Sync When:**

- Simple CRUD operations
- Low traffic services
- CPU-bound workloads
- Team unfamiliar with async/await

### Implementation Example:

```python
# django_service/grpc_server.py
from grpc import aio  # Async gRPC

class AsyncUserServicer(user_pb2_grpc.UserServiceServicer):
    async def CreateUser(
        self,
        request: user_pb2.CreateUserRequest,
        context: aio.ServicerContext
    ) -> user_pb2.User:
        # Use Django's async ORM (Django 4.1+)
        user = await User.objects.acreate(
            name=request.name,
            email=request.email
        )
        return user_pb2.User(
            id=str(user.id),
            name=user.name,
            email=user.email
        )

async def serve():
    server = aio.server()
    user_pb2_grpc.add_UserServiceServicer_to_server(
        AsyncUserServicer(), server
    )
    server.add_insecure_port('[::]:50051')
    await server.start()
    await server.wait_for_termination()

if __name__ == '__main__':
    import asyncio
    asyncio.run(serve())
```

## Troubleshooting

### Issue: "Stub file not found for grpc_tools"

**Solution:** This is expected! `grpc_tools` is a build tool without type stubs. The updated script doesn't import it directly - it uses subprocess.

### Issue: Pylance still shows errors

1. Check Python interpreter is correct:

   ```
   Cmd/Ctrl + Shift + P → "Python: Select Interpreter"
   → Choose: backend (conda)
   ```

2. Verify stub files exist:

   ```bash
   ls django_service/webhooks/grpc_generated/*.pyi
   ```

3. Check Pylance is using stubs:
   ```
   Cmd/Ctrl + Shift + P → "Python: Restart Language Server"
   ```

### Issue: mypy errors differ from Pylance

This is normal - they're different type checkers:

- **Pylance** (VSCode) - Uses Pyright, configured via `pyrightconfig.json`
- **mypy** (CLI) - Configured via `pyproject.toml`

Run both:

```bash
# Check with mypy
mypy django_service/webhooks/grpc_servicers/

# Pylance checks automatically in VSCode
```

## Type Checking Best Practices

### 1. Import Type Stubs Explicitly

```python
# ✅ Good - Types resolved from .pyi
from django_service.webhooks.grpc_generated import (
    user_pb2,
    user_pb2_grpc,
)

# ❌ Bad - Dynamic imports confuse type checker
import importlib
module = importlib.import_module("user_pb2")
```

### 2. Use Type Annotations

```python
# ✅ Good
def CreateUser(
    self,
    request: user_pb2.CreateUserRequest,
    context: grpc.ServicerContext,
) -> user_pb2.User:
    pass

# ❌ Bad
def CreateUser(self, request, context):
    pass
```

### 3. Check Generated Stub Quality

```bash
# View generated stubs
cat django_service/webhooks/grpc_generated/user_pb2.pyi
```

Should contain proper type hints:

```python
class User(Message):
    id: str
    name: str
    email: str
    def __init__(
        self,
        *,
        id: str = ...,
        name: str = ...,
        email: str = ...,
    ) -> None: ...
```

## Configuration Summary

| Tool    | Config File                                    | Purpose              |
| ------- | ---------------------------------------------- | -------------------- |
| Pylance | `pyrightconfig.json` + `.vscode/settings.json` | VSCode type checking |
| mypy    | `pyproject.toml` `[tool.mypy]`                 | CLI type checking    |
| Black   | `pyproject.toml` `[tool.black]`                | Code formatting      |
| isort   | `pyproject.toml` `[tool.isort]`                | Import sorting       |

## Next Steps

1. **Regenerate stubs**: `make proto-stubs-all`
2. **Reload VSCode**: Developer: Reload Window
3. **Test type checking**: Open a servicer file
4. **Run mypy**: `mypy django_service/`
5. **Consider async**: If you have high-traffic endpoints

## Resources

- [gRPC Python Async Documentation](https://grpc.io/docs/languages/python/async/)
- [Pylance Type Checking](https://github.com/microsoft/pylance-release)
- [mypy-protobuf](https://github.com/nipunn1313/mypy-protobuf)
