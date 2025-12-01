## Part 2: Environment Setup (Step-by-Step)

Let's get your development environment ready!

### Step 1: Create Conda Environment

```bash
# Create new environment with Python 3.11
conda create -n webhook-processor python=3.11 -y
conda activate webhook-processor

# Verify
python --version  # Should show 3.11.x
```

---

### Step 2: Project Structure (Monorepo)

```bash
# Create project directory
mkdir webhook-processor
cd webhook-processor

# Create structure
mkdir -p {fastapi_service,django_service,demo_portal,docker}
touch README.md .env.example docker-compose.yml .gitignore
```

**Final structure**:

```
webhook-processor/
├── fastapi_service/          # FastAPI app (port 8000)
│   ├── main.py
│   ├── routers/
│   ├── services/
│   └── requirements.txt
├── django_service/           # Django app (port 8001)
│   ├── manage.py
│   ├── config/              # Django settings
│   ├── webhooks/            # Django app
│   └── requirements.txt
├── demo_portal/             # Static HTML/JS
│   └── index.html
├── docker/                  # Dockerfiles
├── docker-compose.yml
├── .env.example
├── .gitignore
└── README.md
```

---

### Step 3: Install Django + Dependencies

```bash
# Install Django and related packages
pip install django==5.0.0 \
            djangorestframework==3.14.0 \
            psycopg2-binary==2.9.9 \
            django-cors-headers==4.3.0 \
            cryptography==41.0.7 \
            python-decouple==3.8

# Verify FastAPI (you said you have it)
pip list | grep fastapi

# If not installed:
pip install fastapi==0.109.0 \
            uvicorn[standard]==0.27.0 \
            httpx==0.26.0 \
            pydantic==2.5.0

# Save all dependencies
pip freeze > requirements.txt
```

---

### Step 4: Initialize Django Project

```bash
cd django_service

# Create Django project (config is the settings folder name)
django-admin startproject config .

# Create webhooks app
python manage.py startapp webhooks

# Verify structure
ls -la
# Should see: manage.py, config/, webhooks/
```

**Your django_service should now look like**:

```
django_service/
├── manage.py
├── config/
│   ├── __init__.py
│   ├── settings.py      # Main settings file
│   ├── urls.py
│   ├── asgi.py
│   └── wsgi.py
└── webhooks/            # Our app
    ├── __init__.py
    ├── models.py        # Database models
    ├── admin.py         # Admin interface
    ├── views.py
    ├── apps.py
    └── migrations/
```

---

### Step 5: Configure Django Settings

Open `django_service/config/settings.py` and update:

```python
# Around line 20 - add decouple for env variables
from decouple import config

# Around line 28
ALLOWED_HOSTS = ['localhost', '127.0.0.1', '*']  # For development

# Around line 33 - Add your app
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # Third party
    'rest_framework',
    'corsheaders',

    # Local
    'webhooks',
]

# Around line 45 - Add CORS middleware
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'corsheaders.middleware.CorsMiddleware',  # Add this
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

# Around line 80 - Update database (we'll use SQLite for now, PostgreSQL later)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}

# Add at the bottom
CORS_ALLOW_ALL_ORIGINS = True  # For development only

# Encryption key for secrets (add at bottom)
SECRET_ENCRYPTION_KEY = config('SECRET_ENCRYPTION_KEY', default='temp-key-generate-real-one')
```

---

### Step 6: Create .env File

In the **root directory** (webhook-processor/):

```bash
# Create .env file
cat > .env << 'EOF'
# Django
DJANGO_SECRET_KEY=django-insecure-change-this-in-production
DEBUG=True

# Encryption for webhook secrets
SECRET_ENCRYPTION_KEY=generate-this-with-fernet

# Database (for later when we switch to PostgreSQL)
DB_NAME=webhook_db
DB_USER=webhook_user
DB_PASSWORD=change-this-password
DB_HOST=localhost
DB_PORT=5432

# FastAPI
FASTAPI_HOST=0.0.0.0
FASTAPI_PORT=8000

# Django
DJANGO_HOST=0.0.0.0
DJANGO_PORT=8001
EOF
```

---

### Step 7: Test Django Setup

```bash
cd django_service

# Run migrations (creates database tables)
python manage.py migrate

# Create superuser for admin access
python manage.py createsuperuser
# Username: admin
# Email: admin@example.com
# Password: admin123 (or whatever you prefer)

# Start Django server
python manage.py runserver 8001
```

**Open browser**: http://localhost:8001/admin

- Login with credentials you just created
- You should see Django admin interface! 🎉

---

### Step 8: Setup FastAPI Service

```bash
cd ../fastapi_service  # Go back to root, then into fastapi_service

# Create main.py
cat > main.py << 'EOF'
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI(
    title="Webhook Processor API",
    description="Receives and validates third-party webhooks",
    version="1.0.0"
)

# CORS for demo portal
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # In production, specify exact origins
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.get("/")
def read_root():
    return {"message": "Webhook Processor API", "status": "online"}

@app.get("/health")
def health_check():
    return {"status": "healthy"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
EOF
```

**Test it**:

```bash
python main.py
```

**Open browser**: http://localhost:8000

- Should see: `{"message": "Webhook Processor API", "status": "online"}`
- Try: http://localhost:8000/docs (automatic API documentation!)

---

### Step 9: Create .gitignore

In root directory:

```bash
cat > .gitignore << 'EOF'
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
venv/
env/

# Django
*.log
db.sqlite3
db.sqlite3-journal
/media
/staticfiles

# Environment variables
.env
.env.local

# IDEs
.vscode/
.idea/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db
EOF
```

---

## Verification Checklist

Run these commands to verify everything works:

```bash
# Check conda environment
conda info --envs  # Should show webhook-processor

# Check Python packages
pip list | grep -E "django|fastapi|uvicorn"

# Test Django
cd django_service
python manage.py check  # Should show "System check identified no issues"

# Test FastAPI
cd ../fastapi_service
python -c "from main import app; print('FastAPI loads successfully')"
```

---

## What We've Accomplished

✅ Conda environment created
✅ Monorepo structure established
✅ Django installed and configured
✅ FastAPI service created
✅ Admin interface accessible
✅ Both services can run simultaneously (different ports)

---

## Next Steps

Now we're ready to build! In order:

1. **Create Django models** (WebhookProvider, WebhookLog)
2. **Register models in admin** (make them visible in Django Admin)
3. **Add encryption to secret storage** (implement Fernet)
4. **Build FastAPI webhook endpoint** (receive webhooks)
5. **Connect FastAPI to Django DB** (read secrets for validation)

**Should we start with Django models next?** I'll guide you through creating the models with encryption built-in from the start.

Also - did both services start successfully? Any errors?
