Here’s a clean **Markdown documentation** you can drop into your repo (e.g. `docs/common_installation.md`) that explains the setup, commands, and usage with service‑specific examples:

````markdown
# Common Package Installation & Usage

This document explains how to install and use the **`common`** package across all services (`*_service` folders such as `django_service`, `fastapi_service`, etc.).

---

## 📦 Installation Setup

### 1. `pyproject.toml` at Repo Root

We define `common` as an installable package and declare required environment variables per service:

```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "common-utils"
version = "0.1.0"
description = "Shared utilities for Django, FastAPI, and other services"
authors = [{ name = "Your Name", email = "you@example.com" }]
readme = "README.md"
requires-python = ">=3.10"

dependencies = ["python-dotenv"]

[tool.setuptools.packages.find]
where = ["."]
include = ["common*"]

[tool.common.required_vars]
django_service = [
    "GRPC_SERVER_PORT",
    "GRPC_SERVER_CERT_PATH",
    "GRPC_SERVER_KEY_PATH",
    "GRPC_CA_PATH",
]

fastapi_service = [
    "SERVICE_NAME",
    "LOG_LEVEL",
    "APP_PORT",
    "APP_HOST",
    "GRPC_SERVER_ADDRESS",
    "GRPC_CERT_PATH",
    "GRPC_KEY_PATH",
    "GRPC_CA_PATH",
]
```
````

---

### 2. Installation Script

`scripts/install_common.sh` installs `common` into services:

```bash
#!/usr/bin/env bash
set -euo pipefail

ROOT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"

install_common_in_service() {
  local service="$1"
  if [ -d "$ROOT_DIR/$service" ]; then
    echo "→ Installing common into $service"
    pip install -e "$ROOT_DIR"
  else
    echo "❌ Service folder not found: $service"
    exit 1
  fi
}

if [ $# -eq 0 ]; then
  echo "Installing common package into all *_service folders..."
  for service in "$ROOT_DIR"/*_service; do
    if [ -d "$service" ]; then
      install_common_in_service "$(basename "$service")"
    fi
  done
  echo "✅ Common installed in all services"
else
  for service in "$@"; do
    install_common_in_service "$service"
  done
  echo "✅ Common installed in selected services: $*"
fi
```

Make it executable:

```bash
chmod +x scripts/install_common.sh
```

---

### 3. Makefile Targets

Add these targets to your `Makefile`:

```makefile
.PHONY: install-common install-common-django_service install-common-fastapi_service

install-common:
	@./scripts/install_common.sh

install-common-django_service:
	@./scripts/install_common.sh django_service

install-common-fastapi_service:
	@./scripts/install_common.sh fastapi_service
```

Run:

```bash
make install-common              # install into all services
make install-common-django_service  # install into Django only
make install-common-fastapi_service # install into FastAPI only
```

---

## 🛠️ Usage in Services

### Example: Django Service

File: **`django_service/grpc_server.py`**

```python
from pathlib import Path
from common.utils.load_and_validate_env import load_and_validate_env, get_required_vars

service_name = Path(__file__).parent.name  # auto-detects "django_service"
required = get_required_vars(service_name)

env_mode, env_values = load_and_validate_env(Path(__file__).parent, required_vars=required)
```

---

### Example: FastAPI Service

File: **`fastapi_service/main.py`**

```python
from pathlib import Path
from common.utils.load_and_validate_env import load_and_validate_env, get_required_vars

service_name = Path(__file__).parent.name  # auto-detects "fastapi_service"
required = get_required_vars(service_name)

env_mode, env_values = load_and_validate_env(Path(__file__).parent, required_vars=required)
```

---

## 🚀 Workflow

1. Define required environment variables per service in `pyproject.toml`.
2. Run installation:
   - `make install-common` → all services
   - `make install-common-django_service` → Django only
   - `make install-common-fastapi_service` → FastAPI only
3. Import and use `common` utilities in service code.
4. Add new services by:
   - Creating a folder ending with `_service`.
   - Adding required vars in `pyproject.toml`.
   - Adding a Makefile target if single‑service install is desired.

---

✅ With this setup, `common` is reusable, installable, and automatically adapts to any new service you add.

```

Would you like me to also add a **section showing how to extend `pyproject.toml` when a new service (e.g. `billing_service`) is added**, so the workflow is crystal clear for future contributors?
```
