# Professional GitHub Issue & Pull Request Templates

## 📋 Overview

This guide covers three professional templates for different scenarios:

1. **Bug Fix Template** - For fixing existing issues
2. **Feature Implementation Template** - For adding new functionality
3. **Refactor/Enhancement Template** - For improving existing code

---

## 🐛 Template 1: Bug Fix

### When to Use

- Fixing crashes, errors, or incorrect behavior
- Resolving type safety issues
- Patching security vulnerabilities
- Correcting logic errors

### Issue Template

````markdown
## 🐛 Bug Report

### Description

Brief description of the bug and its impact.

### Current Behavior

What actually happens (include error messages, stack traces).

### Expected Behavior

What should happen instead.

### Steps to Reproduce

1. Step one
2. Step two
3. Step three

### Environment

- Python version: 3.11.14
- Framework version: FastAPI 0.x.x
- OS: Ubuntu 22.04

### Code Sample

```python
# Minimal code that reproduces the issue
def problematic_function():
    # This throws AttributeError
    request.client.host  # request.client can be None
```

### Proposed Solution

Brief description of how you plan to fix it.

### Additional Context

Screenshots, logs, or related issues.
````

### Pull Request Template

````markdown
## 🐛 Fix: [Brief description of what was fixed]

### Problem

Closes #[issue-number]

The `request.client` attribute could be `None`, causing `AttributeError` when accessing `.host`. This occurred when [specific scenario].

### Changes Made

- ✅ Added null check for `request.client` before accessing `.host`
- ✅ Added type guard: `if request and request.client`
- ✅ Improved error context logging with additional request metadata
- ✅ Added default values for optional request fields

### Technical Details

**Before:**

```python
"client_ip": request.client.host if request else None,
```

**After:**

```python
"client_ip": request.client.host if request and request.client else None,
```

### Testing

- [x] Tested with normal requests (client present)
- [x] Tested with requests where client is None
- [x] Verified error logging works correctly
- [x] All existing tests pass

### Type Safety

- [x] Pylance strict mode passes
- [x] No type: ignore comments needed

### Checklist

- [x] Code follows project style guidelines
- [x] Self-review completed
- [x] Comments added for complex logic
- [x] Documentation updated
- [x] No breaking changes
````

---

## ✨ Template 2: Feature Implementation

### When to Use

- Adding new endpoints or functionality
- Implementing new features requested by users
- Adding new utilities or helpers
- Expanding API capabilities

### Issue Template

````markdown
## ✨ Feature Request

### Problem Statement

What problem does this feature solve? Who benefits from it?

### Proposed Solution

Describe the feature and how it should work.

### Example Usage

```python
# How developers would use this feature
result = new_feature(param1, param2)
```

### Alternative Solutions

Other approaches you considered and why you chose this one.

### Dependencies

- New libraries needed
- Breaking changes
- Migration requirements

### Success Criteria

- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Documentation complete
````

### Pull Request Template

````markdown
## ✨ Feature: [Feature name]

### Overview

Closes #[issue-number]

Implements [feature name] to solve [problem]. This feature allows users to [benefit].

### Implementation Details

#### New Components

- **`APIError` class**: Base exception with centralized logging
- **Error handler**: Global FastAPI error handler
- **Convenience functions**: `validation_error()`, `not_found_error()`, etc.

#### Key Features

1. **Automatic logging**: Errors are logged with full context automatically
2. **Structured error responses**: Consistent JSON format with error IDs
3. **Request tracking**: Captures request path, method, IP, and user-agent
4. **Type-safe**: Full type hints and Pylance strict compliance

### Code Examples

**Basic usage:**

```python
@app.get("/users/{user_id}")
async def get_user(user_id: int):
    user = await db.get_user(user_id)
    if not user:
        raise not_found_error(f"User {user_id} not found")
    return user
```

**With exception details:**

```python
try:
    result = await external_api.call()
except Exception as e:
    raise server_error("External API failed", detail=e)
```

### API Changes

#### New Exports

- `APIError` - Base exception class
- `api_error_handler` - FastAPI error handler
- `validation_error()`, `not_found_error()`, `unauthorized_error()`
- `forbidden_error()`, `conflict_error()`, `server_error()`
- `service_unavailable_error()`

#### Response Format

```json
{
  "message": "User 123 not found",
  "status_code": 404,
  "error_code": "NOT_FOUND",
  "error_id": "20241203143022"
}
```

### Testing Strategy

- [x] Unit tests for all error types
- [x] Integration tests with FastAPI app
- [x] Error logging verification
- [x] Request context capture tests
- [x] Edge cases (None values, missing fields)

### Documentation

- [x] Docstrings for all public functions
- [x] Usage examples in README
- [x] API documentation updated
- [x] Migration guide (if needed)

### Performance Impact

- Minimal: Only called on error paths
- No impact on happy path performance
- Logging is async-friendly

### Breaking Changes

- [ ] None - fully backward compatible

### Checklist

- [x] Code follows style guidelines
- [x] Type hints complete (Pylance strict)
- [x] Tests written and passing
- [x] Documentation updated
- [x] Self-review completed
- [x] No security vulnerabilities introduced
````

---

## 🔧 Template 3: Refactor/Enhancement

### When to Use

- Improving code quality without changing functionality
- Performance optimizations
- Better error handling
- Type safety improvements
- Code organization improvements

### Issue Template

```markdown
## 🔧 Code Quality Enhancement

### Current State

Describe what exists now and why it needs improvement.

### Problems Identified

- Problem 1: [Description]
- Problem 2: [Description]
- Problem 3: [Description]

### Proposed Improvements

1. **Improvement 1**: [Details]
2. **Improvement 2**: [Details]

### Benefits

- Better type safety
- Improved maintainability
- Clearer code structure
- [Other benefits]

### Impact

- [ ] No breaking changes
- [ ] No functionality changes
- [ ] Performance improvement: [metrics]

### Related Issues

Links to related issues or discussions.
```

### Pull Request Template

````markdown
## 🔧 Refactor: [What was improved]

### Motivation

The existing code had several type safety issues when running Pylance in strict mode. This refactor improves type safety and code quality without changing functionality.

### Changes Summary

#### Type Safety Improvements

- ✅ Added proper type hints for all functions and methods
- ✅ Fixed `Optional` type handling
- ✅ Resolved all Pylance strict mode errors
- ✅ Changed `Dict` to `Dict[str, Any]` with proper imports
- ✅ Added null safety checks for optional attributes

#### Code Quality Improvements

- ✅ Better exception handling (no bare `except:`)
- ✅ Explicit type annotations for complex types
- ✅ Improved error messages and context
- ✅ Better docstrings and comments

#### Detailed Changes

**1. Fixed `request.client` null safety:**

```python
# Before
"client_ip": request.client.host if request else None

# After
"client_ip": request.client.host if request and request.client else None
```

**2. Added proper type imports:**

```python
from typing import Optional, Any, Dict
```

**3. Enhanced error context:**

```python
context: Dict[str, Any] = {
    "timestamp": datetime.now(timezone.utc).isoformat(),
    "status_code": self.status_code,
    "error_code": self.error_code,
    "error_id": self.error_id,
    "request_method": request.method if request else None,  # NEW
    "user_agent": request.headers.get("user-agent") if request else None,  # NEW
}
```

### Before/After Metrics

- Pylance errors: 8 → 0
- Type coverage: 85% → 100%
- Code complexity: No change
- Performance: No change

### Testing

- [x] All existing tests pass
- [x] No functionality changes
- [x] Type checking passes (Pylance strict)
- [x] Manual testing completed

### Non-Goals

This PR intentionally does NOT:

- Change functionality
- Add new features
- Modify API contracts
- Affect performance

### Backward Compatibility

✅ Fully backward compatible - zero breaking changes

### Checklist

- [x] Code follows style guidelines
- [x] All Pylance strict errors resolved
- [x] Tests pass
- [x] No functionality changes
- [x] Documentation updated (if needed)
- [x] Self-review completed

```

```
````

---

## 🎯 Quick Decision Guide

### Choose Bug Fix Template When:

- ❌ Something is broken or throwing errors
- ❌ Security vulnerability discovered
- ❌ Type errors preventing compilation
- ❌ Incorrect behavior in production

### Choose Feature Template When:

- ✨ Adding new capabilities
- ✨ Implementing user requests
- ✨ Creating new utilities
- ✨ Expanding API surface

### Choose Refactor Template When:

- 🔧 Improving code quality
- 🔧 Fixing type safety issues (no bugs)
- 🔧 Optimizing performance
- 🔧 Reorganizing code structure
- 🔧 Adding better error handling

---

## 💡 Best Practices

### General Tips

1. **Be specific**: "Fixed null pointer" > "Fixed bug"
2. **Show, don't tell**: Include code examples
3. **Link context**: Reference related issues/PRs
4. **Explain why**: Not just what, but why this solution
5. **Consider reviewers**: Make their job easy

### Issue Writing

- Write issues you'd want to receive
- Include reproduction steps
- Provide expected vs actual behavior
- Add error messages and stack traces
- Tag appropriately (bug, enhancement, etc.)

### Pull Request Writing

- Start with the problem, not the solution
- Explain your reasoning
- Show before/after comparisons
- Include test evidence
- Make reviewing easy with clear structure

### Common Mistakes to Avoid

- ❌ Vague titles: "fix stuff" or "updates"
- ❌ No context: Diving into code without explaining why
- ❌ Missing tests: Claiming it works without proof
- ❌ Scope creep: Fixing 10 unrelated things in one PR
- ❌ No self-review: Not checking your own work first

---

## 📚 Example: This api_error.py Fix

For the `api_error.py` fix, you would use the **Bug Fix Template**:

**Issue Title:** `🐛 AttributeError: 'NoneType' object has no attribute 'host' in api_error.py`

**PR Title:** `🐛 Fix: Add null safety check for request.client in APIError logging`

This is a bug fix because:

- It fixes a runtime error (AttributeError)
- It's a type safety issue caught by Pylance
- It could cause crashes in production
- The fix is targeted and specific

---

## 🚀 Bonus: Commit Message Format

Follow conventional commits:

```

<type>(<scope>): <subject>

<body>

<footer>
```

**Types:**

- `fix:` - Bug fixes
- `feat:` - New features
- `refactor:` - Code changes without functionality changes
- `docs:` - Documentation changes
- `test:` - Test changes
- `chore:` - Build/config changes

**Example:**

```
fix(api-error): add null safety check for request.client

- The request.client attribute can be None in certain FastAPI scenarios,
  causing AttributeError when accessing .host.

- Added proper null check to handle this case gracefully.

Fixes #123
```

---

**Remember:** Good documentation is as important as good code. These templates help communicate your work effectively to your team and future self! 🎉
