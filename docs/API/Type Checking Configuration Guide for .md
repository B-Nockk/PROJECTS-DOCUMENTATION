# Type Checking Configuration Guide for gRPC Services

## Problem Summary

When working with gRPC generated code in strict type checking mode (Pylance/Pyright), you encounter issues because:

1. **Generated `.py` files have incomplete types** - protoc generates working Python code but with `Any` types everywhere
2. **Generated `.pyi` stub files also have incomplete types** - protobuf messages use `Mapping[Unknown, Unknown]` for nested messages
3. **Pylance analyzes both `.py` and `.pyi` files** by default, causing conflicts and errors
4. **context.abort() never returns** but type checkers don't know this, so they complain about missing returns

## Solution Strategy

### 1. **Exclude Generated Code from Analysis**

The key insight: Don't try to type-check generated protobuf code. Instead:

- Completely exclude `grpc_generated/` directories from Pylance analysis
- Disable unknown type warnings globally (since protobuf types are inherently partial)
- Use the `.pyi` stubs for type hints but don't analyze them

### 2. **Proper Type Annotations in Servicer Code**

Your servicer implementations need explicit types:

```python
def CreateOrganization(
    self,
    request: organization_pb2.CreateOrganizationRequest,  # ✅ Explicit type
    context: grpc.ServicerContext,                         # ✅ Explicit type
) -> organization_pb2.Organization:                        # ✅ Return type
```

**NOT:**

```python
def CreateOrganization(self, request, context):  # ❌ No types
```

### 3. **Handle context.abort() Properly**

`context.abort()` raises an exception and never returns, but type checkers don't know this. Solution:

```python
try:
    org = Organization.objects.get(id=request.id)
    return self._organization_to_proto(org)
except Organization.DoesNotExist:
    context.abort(grpc.StatusCode.NOT_FOUND, f"Not found")
    return organization_pb2.Organization()  # ✅ Never reached but satisfies type checker
```

### 4. **Cast Protobuf Values to Python Types**

Protobuf fields return special descriptor types. Cast them explicitly:

```python
# ❌ This might show warnings
org.name = request.name

# ✅ Explicit cast
org.name = str(request.name)
org.webhook_limit = int(request.webhook_limit)
```

## Configuration Files

### `.vscode/settings.json`

Key changes:

```json
{
  // Exclude generated code completely
  "python.analysis.exclude": ["**/grpc_generated/**"],

  // Disable library code analysis
  "python.analysis.useLibraryCodeForTypes": false,

  // Suppress unknown type warnings globally
  "python.analysis.diagnosticSeverityOverrides": {
    "reportUnknownParameterType": "none",
    "reportUnknownArgumentType": "none"
  }
}
```

### `pyrightconfig.json`

Key changes:

```json
{
  "exclude": [
    "**/grpc_generated" // Don't analyze generated code
  ],
  "reportUnknownParameterType": "none", // Suppress globally
  "useLibraryCodeForTypes": false
}
```

### `pyproject.toml` - mypy config

```toml
[tool.mypy]
# Skip generated proto files but use their .pyi stubs
[[tool.mypy.overrides]]
module = [
  "django_service.webhooks.grpc_generated.*",
  "fastapi_service.grpc_generated.*",
]
follow_imports = "skip"
ignore_errors = true
ignore_missing_imports = false  # Still enforce stub types
```

## Step-by-Step Setup

### 1. Update Configuration Files

```bash
# Copy the fixed configurations from the artifacts above
cp .vscode/settings.json.new .vscode/settings.json
cp pyrightconfig.json.new pyrightconfig.json
```

### 2. Regenerate Proto Files with py.typed Markers

```bash
# Updated proto_tools.py now creates py.typed automatically
make proto-stubs-all

# Or manually create markers
chmod +x scripts/create_py_typed.sh
./scripts/create_py_typed.sh
```

### 3. Reload VS Code

```
Cmd/Ctrl + Shift + P → "Developer: Reload Window"
```

### 4. Update All Servicer Files

For each servicer method:

**Before:**

```python
def CreateUser(self, request, context):  # ❌
    user = User.objects.create(
        email=request.email,  # ⚠️  Unknown type warning
        name=request.name,
    )
    return self._user_to_proto(user)  # ❌ Missing return warning
```

**After:**

```python
def CreateUser(
    self,
    request: user_pb2.CreateUserRequest,  # ✅ Typed
    context: grpc.ServicerContext,        # ✅ Typed
) -> user_pb2.User:                       # ✅ Return type
    try:
        user = User.objects.create(
            email=str(request.email),      # ✅ Explicit cast
            name=str(request.name),
        )
        return self._user_to_proto(user)
    except Exception as e:
        context.abort(grpc.StatusCode.INTERNAL, str(e))
        return user_pb2.User()  # ✅ Satisfies type checker
```

## Common Patterns

### Pattern 1: HasField() for Optional Fields

```python
def UpdateOrg(
    self,
    request: organization_pb2.UpdateOrganizationRequest,
    context: grpc.ServicerContext,
) -> organization_pb2.Organization:
    org = Organization.objects.get(id=request.id)

    if request.HasField("name"):
        org.name = str(request.name)
    if request.HasField("webhook_limit"):
        org.webhook_limit = int(request.webhook_limit)

    org.save()
    return self._organization_to_proto(org)
```

### Pattern 2: Pagination

```python
def ListUsers(
    self,
    request: user_pb2.ListUsersRequest,
    context: grpc.ServicerContext,
) -> user_pb2.UserList:
    paginator = Paginator(
        request.pagination.page if request.HasField("pagination") else None,
        request.pagination.page_size if request.HasField("pagination") else None,
    )
    # ... rest of implementation
```

### Pattern 3: Error Handling

```python
def GetUser(
    self,
    request: user_pb2.IdRequest,
    context: grpc.ServicerContext,
) -> user_pb2.User:
    try:
        user = User.objects.get(id=request.id)
        return self._user_to_proto(user)
    except User.DoesNotExist:
        context.abort(grpc.StatusCode.NOT_FOUND, f"User {request.id} not found")
        return user_pb2.User()  # Never reached
    except Exception as e:
        context.abort(grpc.StatusCode.INTERNAL, str(e))
        return user_pb2.User()  # Never reached
```

## Verification

After applying changes, verify everything works:

```bash
# 1. Check Pylance in VS Code
# Open any servicer file - should see no red squiggles

# 2. Run mypy
mypy django_service/webhooks/grpc_servicers/

# 3. Run pyright explicitly
pyright django_service/webhooks/grpc_servicers/

# 4. Generate all protos fresh
make proto-clean
make proto-stubs-all
```

## Why This Works

1. **Exclusion prevents analysis** - Pylance never tries to type-check the incomplete generated code
2. **Stubs provide hints** - The `.pyi` files still give you autocomplete and basic type info
3. **Explicit types in servicers** - Your code has full type safety where it matters
4. **Global suppression** - Unknown types from protobuf are expected and suppressed globally
5. **py.typed markers** - Tell type checkers these packages have stubs (even if incomplete)

## Troubleshooting

### "Still seeing warnings in .pyi files"

The `.pyi` files themselves might show warnings in VS Code, but:

- These don't affect your servicer code
- You shouldn't edit `.pyi` files anyway
- The warnings are cosmetic - they don't break type checking

### "context.abort() still shows missing return"

Make sure you added the unreachable return statement:

```python
context.abort(...)
return SomeMessage()  # This line is required
```

### "Import errors in servicer files"

Check your `extraPaths` in VS Code settings includes the grpc_generated folders:

```json
"python.analysis.extraPaths": [
  "${workspaceFolder}/django_service/webhooks/grpc_generated"
]
```

### "After reload, still seeing issues"

1. Close all Python files
2. Reload VS Code window
3. Let Pylance fully re-index (watch bottom status bar)
4. Open your servicer file again

## Summary

The key insight is: **Don't fight with generated protobuf code's incomplete types**. Instead:

- Exclude it from analysis
- Accept that protobuf types are partial
- Focus type safety on YOUR code (servicers, models, utilities)
- Use explicit type annotations everywhere in your servicers

This gives you the best of both worlds:

- Full type safety in your business logic
- No noise from generated code
- Autocomplete and basic hints from stubs
- Clean, maintainable code
