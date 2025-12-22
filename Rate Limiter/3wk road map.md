Perfect - thinking about future growth is good instincts, but starting simple is the right call. You can always refactor later and show that evolution in your git history.

Let me collate everything into one clean 3-week roadmap now that we've locked in the tech stack.

---

# 3-Week DevOps Job-Ready Roadmap

## Tech Stack (Locked In)
- **Week 1 Project:** FastAPI + Django Admin + PostgreSQL + Redis + Docker
- **Architecture:** Shared database (simple, can evolve to gRPC later)
- **Week 2:** Add CI/CD (GitHub Actions) + Deploy to cloud
- **Week 3:** Add Prometheus + Grafana monitoring

---

## WEEK 1: API Rate Limiter with Admin Panel

### Day 1: Foundations (3-4 hours)
**Linux Essentials (1.5 hours)**
- File operations: `ls`, `cd`, `pwd`, `mkdir`, `rm`, `cp`, `mv`
- Text processing: `grep`, `awk`, `sed`, `tail -f` (for logs)
- Processes: `ps`, `top`, `kill`, `systemctl`
- Networking: `curl`, `netstat`/`ss`, `ping`
- Permissions: `chmod`, `chown`

**Practice:** Run a Flask app locally, tail logs, kill and restart process

**Docker Basics (1.5-2 hours)**
- Containers vs images
- Commands: `docker run`, `docker build`, `docker ps`, `docker logs`, `docker exec`
- Write a simple Dockerfile
- docker-compose: services, networks, volumes

**Practice:** Containerize a "Hello World" app, run it

---

### Day 2: Git + More Docker (3-4 hours)
**Git Workflows (1.5 hours)**
- `clone`, `add`, `commit`, `push`, `pull`
- Branching: `git checkout -b feature`, `git merge`
- .gitignore, commit messages
- Push to GitHub

**Docker Deep Dive (1.5-2 hours)**
- Multi-stage builds
- Environment variables
- Volume mounts
- docker-compose with multiple services
- Networking between containers

**Practice:** Build a docker-compose with Python app + PostgreSQL, test connection

---

### Day 3: FastAPI Service (4-5 hours)

**Setup:**
```
fastapi-rate-limiter/
├── fastapi-service/
│   ├── main.py
│   ├── models.py (SQLAlchemy models)
│   ├── rate_limiter.py (Redis logic)
│   ├── requirements.txt
│   └── Dockerfile
```

**Build:**
- FastAPI app with endpoints:
  - `POST /api/keys` - Generate new API key
  - `POST /api/request` - Make rate-limited request (requires X-API-Key header)
  - `GET /api/check` - Check quota remaining
  - `GET /health` - Health check
- SQLAlchemy models: `APIKey`, `RequestLog`
- Redis rate limiting logic (token bucket or sliding window)
- Return 429 status when rate limited

**Test:** Use curl to hit endpoints, verify rate limiting works

---

### Day 4: Django Admin Service (4-5 hours)

**Setup:**
```
fastapi-rate-limiter/
├── django-admin/
│   ├── manage.py
│   ├── config/
│   │   ├── settings.py
│   │   └── urls.py
│   ├── api_manager/
│   │   ├── models.py (same schema as FastAPI)
│   │   ├── admin.py
│   │   └── apps.py
│   ├── requirements.txt
│   └── Dockerfile
```

**Build:**
- Django models matching FastAPI schema
- Admin configuration:
  ```python
  @admin.register(APIKey)
  class APIKeyAdmin(admin.ModelAdmin):
      list_display = ['key', 'tier', 'requests_per_hour', 'is_active', 'created_at']
      list_filter = ['tier', 'is_active']
      search_fields = ['key']
      readonly_fields = ['created_at']
      
      def save_model(self, request, obj, form, change):
          if not change:  # New object
              obj.key = secrets.token_urlsafe(32)
          super().save_model(request, obj, form, change)
  
  @admin.register(RequestLog)
  class RequestLogAdmin(admin.ModelAdmin):
      list_display = ['api_key', 'endpoint', 'status_code', 'timestamp']
      list_filter = ['status_code', 'timestamp']
      date_hierarchy = 'timestamp'
  ```

**Test:** Run Django, create superuser, access admin panel at `/admin`

---

### Day 5: Integration + Docker Compose (4-5 hours)

**Create docker-compose.yml:**
```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: ratelimiter
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  fastapi:
    build: ./fastapi-service
    ports:
      - "8000:8000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://admin:password@postgres:5432/ratelimiter
      REDIS_URL: redis://redis:6379/0

  django:
    build: ./django-admin
    ports:
      - "8001:8000"
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://admin:password@postgres:5432/ratelimiter
      DJANGO_SUPERUSER_USERNAME: admin
      DJANGO_SUPERUSER_PASSWORD: admin
      DJANGO_SUPERUSER_EMAIL: admin@example.com
    command: >
      sh -c "python manage.py migrate &&
             python manage.py createsuperuser --noinput || true &&
             python manage.py runserver 0.0.0.0:8000"

volumes:
  postgres_data:
```

**Test:**
```bash
docker-compose up --build
# Access Django Admin: http://localhost:8001/admin
# Test FastAPI: curl http://localhost:8000/health
```

**Fix any issues:** Networking, environment variables, database connections

---

### Day 6: Testing + Polish (4-5 hours)

**Add Testing:**
- Write pytest tests for FastAPI endpoints
- Test rate limiting logic
- Test database operations
- Create a load test script:
  ```python
  # load_test.py
  import requests
  import time
  
  API_KEY = "your-test-key"
  
  for i in range(150):
      response = requests.post(
          "http://localhost:8000/api/request",
          headers={"X-API-Key": API_KEY}
      )
      print(f"Request {i+1}: {response.status_code}")
      time.sleep(0.1)
  ```

**Polish:**
- Add proper error handling
- Add logging (structured logs)
- Create .env.example file
- Add health checks to both services

---

### Day 7: Documentation (3-4 hours)

**Create comprehensive README.md:**
```markdown
# API Rate Limiter with Admin Dashboard

## Architecture
[Include diagram showing: FastAPI ↔ Redis, both ↔ PostgreSQL, Django Admin ↔ PostgreSQL]

## Features
- RESTful rate limiting API
- Redis-backed token bucket algorithm
- Django Admin for key management
- Docker containerized
- PostgreSQL for persistence

## Quick Start
```bash
# Clone repo
git clone [your-repo]

# Start all services
docker-compose up

# Access services
- FastAPI: http://localhost:8000
- Django Admin: http://localhost:8001/admin (admin/admin)
- API Docs: http://localhost:8000/docs
```

## Testing Rate Limiting
```bash
# Create API key in Django Admin first
python load_test.py
```

## Architecture Decisions
### Why FastAPI for public API?
- Async support for high concurrency
- Automatic OpenAPI documentation
- Fast request handling for rate limiting

### Why Django Admin?
- Professional management UI without custom frontend
- Built-in authentication and permissions
- Rapid development

### Why shared database?
- Simple CRUD operations don't require service layer
- Reduces latency (no network hop)
- Can evolve to gRPC if complexity grows

## Future Enhancements
- Distributed rate limiting across multiple instances
- Refactor to gRPC for inter-service communication
- Add analytics dashboard
- Implement tiered pricing automation
```

**Take Screenshots:**
- Django Admin showing API keys list
- Django Admin showing request logs
- FastAPI docs page
- Terminal showing rate limit (429 errors)

**Push to GitHub:**
- Clean commit history
- Proper .gitignore
- All documentation in place

---

## WEEK 2: CI/CD + Cloud Deployment

### Day 8: GitHub Actions Setup (3-4 hours)

**Create .github/workflows/ci.yml:**
```yaml
name: CI Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test-fastapi:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
    - uses: actions/checkout@v3
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    - name: Install dependencies
      run: |
        cd fastapi-service
        pip install -r requirements.txt
        pip install pytest pytest-cov
    - name: Run tests
      run: |
        cd fastapi-service
        pytest --cov=. --cov-report=xml
    - name: Upload coverage
      uses: codecov/codecov-action@v3

  lint:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    - name: Lint with flake8
      run: |
        pip install flake8
        flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
        flake8 . --count --max-complexity=10 --max-line-length=127 --statistics

  build-and-push:
    runs-on: ubuntu-latest
    needs: [test-fastapi, lint]
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v3
    - name: Login to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKERHUB_USERNAME }}
        password: ${{ secrets.DOCKERHUB_TOKEN }}
    - name: Build and push FastAPI
      uses: docker/build-push-action@v4
      with:
        context: ./fastapi-service
        push: true
        tags: yourusername/rate-limiter-api:latest
    - name: Build and push Django
      uses: docker/build-push-action@v4
      with:
        context: ./django-admin
        push: true
        tags: yourusername/rate-limiter-admin:latest
```

**Setup:**
- Create Docker Hub account (free)
- Add secrets to GitHub repo settings
- Push to trigger pipeline
- Verify builds pass

---

### Day 9-10: Cloud Provider Setup (6-8 hours)

**Choose: Oracle Cloud (Always Free Tier)**

**Why Oracle over AWS/GCP:**
- 2 AMD-based VMs (always free, not 12-month trial)
- 200GB block storage
- 10TB outbound data transfer/month
- Actually free forever, not credit-based

**Setup Steps:**

**Day 9: Provision Infrastructure (3-4 hours)**
1. Create Oracle Cloud account
2. Create VPC/Network
3. Provision 2 VMs:
   - VM1: FastAPI + Redis
   - VM2: Django + PostgreSQL
4. Configure security lists (firewall rules):
   - Allow ports 80, 443, 8000, 8001
   - SSH access
5. Set up SSH keys

**Day 10: Manual Deployment (3-4 hours)**
1. SSH into VMs
2. Install Docker + docker-compose
3. Clone your repo
4. Set up environment variables
5. Run docker-compose up
6. Configure reverse proxy (nginx):
   ```nginx
   server {
       listen 80;
       server_name your-domain.com;
       
       location /api {
           proxy_pass http://localhost:8000;
       }
       
       location /admin {
           proxy_pass http://localhost:8001;
       }
   }
   ```
7. Test endpoints publicly
8. (Optional) Set up free SSL with Let's Encrypt

---

### Day 11: Infrastructure as Code (4-5 hours)

**Install Terraform locally**

**Create terraform/ directory:**
```
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
└── terraform.tfvars.example
```

**Write Terraform configs:**
```hcl
# main.tf
terraform {
  required_providers {
    oci = {
      source = "oracle/oci"
    }
  }
}

provider "oci" {
  region = var.region
}

resource "oci_core_instance" "api_server" {
  availability_domain = var.availability_domain
  compartment_id      = var.compartment_id
  shape               = "VM.Standard.E2.1.Micro"
  
  source_details {
    source_type = "image"
    source_id   = var.image_id
  }
  
  create_vnic_details {
    subnet_id = oci_core_subnet.public_subnet.id
  }
  
  metadata = {
    ssh_authorized_keys = file(var.ssh_public_key_path)
    user_data = base64encode(file("cloud-init.sh"))
  }
}

# ... more resources
```

**cloud-init.sh (bootstrap script):**
```bash
#!/bin/bash
# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker ubuntu

# Install docker-compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Clone and run app
git clone [your-repo] /opt/rate-limiter
cd /opt/rate-limiter
docker-compose up -d
```

**Test:**
```bash
terraform init
terraform plan
terraform apply
```

**Document:** Add Terraform instructions to README

---

### Day 12: Monitoring Setup Basics (3-4 hours)

**Add to docker-compose.yml:**
```yaml
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'

volumes:
  prometheus_data:
```

**Create prometheus.yml:**
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'fastapi'
    static_configs:
      - targets: ['fastapi:8000']
```

**Add metrics to FastAPI:**
```python
from prometheus_fastapi_instrumentator import Instrumentator

app = FastAPI()
Instrumentator().instrument(app).expose(app)
```

**Test:** Access Prometheus at http://localhost:9090

---

### Day 13-14: Documentation + Polish (6-8 hours)

**Day 13: Update Documentation (3-4 hours)**
- Update README with CI/CD section
- Document deployment process
- Add architecture diagram with cloud resources
- Create DEPLOYMENT.md with step-by-step cloud setup
- Add troubleshooting section

**Day 14: Portfolio Prep (3-4 hours)**
- Create project demo video (2-3 minutes):
  - Show Django Admin
  - Demo rate limiting with curl
  - Show CI/CD pipeline
  - Show live deployment
- Update GitHub profile README
- Write blog post about what you learned (optional but good)
- Prepare 2-minute elevator pitch for the project

---

## WEEK 3: Advanced Monitoring + Job Applications

### Day 15-16: Grafana Dashboards (6-8 hours)

**Day 15: Grafana Setup (3-4 hours)**

**Add to docker-compose.yml:**
```yaml
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    depends_on:
      - prometheus

volumes:
  grafana_data:
```

**Create grafana/provisioning/datasources/prometheus.yml:**
```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

**Day 16: Build Dashboard (3-4 hours)**
- Create dashboard showing:
  - Request rate (requests/second)
  - Rate limit hits (429 responses)
  - Active API keys
  - Request latency percentiles (p50, p95, p99)
  - Redis connection status
- Export dashboard JSON
- Add to repo: `grafana/dashboards/rate-limiter.json`
- Document how to import dashboard

---

### Day 17: Add Custom Metrics (3-4 hours)

**Enhance FastAPI metrics:**
```python
from prometheus_client import Counter, Histogram, Gauge

# Custom metrics
rate_limit_hits = Counter('rate_limit_hits_total', 'Total rate limit hits', ['api_key_tier'])
active_keys = Gauge('active_api_keys', 'Number of active API keys', ['tier'])
request_duration = Histogram('request_duration_seconds', 'Request duration', ['endpoint'])

# Use in code
@app.post("/api/request")
async def make_request(api_key: str = Header(None)):
    with request_duration.labels(endpoint="/api/request").time():
        # ... rate limiting logic
        if rate_limited:
            rate_limit_hits.labels(api_key_tier=tier).inc()
            raise HTTPException(429)
```

**Update Grafana dashboard** to show new metrics

**Add alerting rules** (optional):
```yaml
# prometheus/alerts.yml
groups:
  - name: rate_limiter
    interval: 30s
    rules:
      - alert: HighRateLimitHits
        expr: rate(rate_limit_hits_total[5m]) > 10
        annotations:
          summary: "High rate of rate limit hits"
```

---

### Day 18-19: Resume + Application Prep (6-8 hours)

**Day 18: Resume & LinkedIn (3-4 hours)**

**Update Resume:**
- Add "DevOps Engineer" or "Backend/DevOps Engineer" to title
- Add skills section:
  ```
  DevOps: Docker, Kubernetes, Terraform, CI/CD (GitHub Actions), 
          Prometheus, Grafana, nginx
  Cloud: Oracle Cloud, AWS (learning)
  Backend: Python, FastAPI, Django, Redis, PostgreSQL
  Tools: Git, Linux, gRPC
  ```
- Add project under experience:
  ```
  API Rate Limiter Platform | Personal Project
  - Designed and deployed microservices architecture with FastAPI and Django
  - Implemented Redis-backed rate limiting handling 1000+ req/sec
  - Built CI/CD pipeline with automated testing and Docker image builds
  - Deployed to Oracle Cloud with Terraform (Infrastructure as Code)
  - Created monitoring dashboards with Prometheus and Grafana
  - Technologies: Docker, FastAPI, Django, Redis, PostgreSQL, Terraform, 
    GitHub Actions, Prometheus, Grafana
  ```

**Update LinkedIn:**
- Same skills additions
- Add project with link to GitHub
- Write post announcing your new project
- Connect with DevOps engineers and recruiters

**Day 19: Job Search Strategy (3-4 hours)**

**Identify Target Companies:**
- Remote-first companies (they're used to international hires)
- Companies in: UK, EU, Canada, US (better payment infrastructure for Nigerian contractors)
- Platforms: LinkedIn, AngelList, We Work Remotely, Remote OK, Turing.com

**Create Target List (10-15 companies):**
- Check if they hire internationally
- Note their tech stack (prefer ones using Python/Docker)
- Find DevOps engineers there on LinkedIn

**Application Materials:**
- Customize cover letter template mentioning your project
- Prepare "Why DevOps?" story
- Practice explaining project in 2 minutes

---

### Day 20: Interview Prep (4-5 hours)

**Technical Prep - Common Questions:**

**Docker/Containers:**
- "Explain the difference between a Docker image and container"
- "How would you debug a container that won't start?"
- "What's a multi-stage Dockerfile and why use it?"

**CI/CD:**
- "Walk me through your CI/CD pipeline"
- "How do you handle secrets in CI/CD?"
- "What's the difference between continuous delivery and deployment?"

**Linux/Networking:**
- "How do you check which process is using a port?"
- "Explain what happens when you type a URL in a browser"
- "How do you tail logs of a specific process?"

**Monitoring:**
- "What metrics would you monitor for a web application?"
- "Explain the difference between logs, metrics, and traces"
- "How would you debug high latency in an API?"

**Your Project:**
- "Walk me through your rate limiter architecture"
- "Why did you choose FastAPI over Django for the API?"
- "How does your rate limiting algorithm work?"
- "How would you scale this to handle 10x traffic?"

**Practice:**
- Record yourself answering these (Loom/phone video)
- Do mock interview with a friend
- Practice explaining your architecture diagram

---

### Day 21: Start Applying (4-5 hours)

**Morning: Apply to 5 companies**
- Customize cover letter for each
- Include link to GitHub project
- Mention you're open to remote work from Nigeria

**Afternoon: Networking**
- Find DevOps engineers at target companies on LinkedIn
- Send connection requests with personalized note:
  ```
  Hi [Name], I'm a backend engineer transitioning to DevOps and recently
  built [project link]. I saw you work at [Company] - would love to learn
  about your experience there. Open to a quick chat if you have time!
  ```

**Evening: Platform Profiles**
- Create/update Turing.com profile (AI-vetted, good for international)
- Update Upwork profile (add DevOps skills)
- Check AngelList for startup DevOps roles

**Set Daily Goal:** 3-5 applications per day moving forward

---

## Post-Week 3: Continuous Improvement

**While Job Hunting:**
- **Week 4:** Add PROJECT 2 - Kubernetes version (use Minikube locally)
  - Convert docker-compose to K8s manifests
  - Deploy with kubectl
  - Add to portfolio
  
- **Week 5:** Learn AWS basics (free tier)
  - Deploy same project to AWS
  - Add AWS to resume
  - Now you have multi-cloud experience

- **Week 6+:** Keep applying, keep learning
  - Learn Ansible (configuration management)
  - Study for certifications (AWS CCP, Terraform Associate)
  - Contribute to open source DevOps tools

---

## Success Metrics

**By End of Week 3, You Should Have:**
- ✅ 2-3 portfolio projects on GitHub
- ✅ Live deployed application with URL
- ✅ CI/CD pipeline that actually runs
- ✅ Monitoring dashboards with real metrics
- ✅ Infrastructure as Code (Terraform)
- ✅ Updated resume and LinkedIn
- ✅ Applied to 10+ jobs
- ✅ Can confidently explain your project in interviews

---

## Daily Time Commitment

- **Weekdays:** 5-6 hours (adjust based on your schedule)
- **Weekends:** 6-8 hours (catch up if needed)
- **Total:** ~40 hours/week (treat it like a full-time job)

---

## Free Resources Summary

**Cloud:**
- Oracle Cloud (always free tier) ← Primary
- Railway.app ($5/month free)
- Render.com (free tier)

**Tools:**
- Docker Desktop (free)
- Terraform (open source)
- GitHub Actions (2000 minutes/month free)
- Docker Hub (free public repos)

**Learning:**
- FreeCodeCamp (DevOps tutorials)
- TechWorld with Nana (YouTube - excellent Docker/K8s)
- Official docs (always the best resource)

---

Is this clear? Any specific day you want me to break down further, or concerns about timeline/difficulty?
