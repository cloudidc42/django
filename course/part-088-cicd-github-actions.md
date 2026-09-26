# Part 088: CI/CD ด้วย GitHub Actions

> **ขั้นตอนที่ 871-880** | Phase 11: DevOps & Deployment

---

## ขั้นตอนที่ 871: GitHub Actions คืออะไร

### 871.1 แนวคิด

**GitHub Actions** คือ CI/CD platform ที่ built-in ใน GitHub ทำงานผ่าน YAML workflows

```
.github/
  workflows/
    ci.yml           # test + lint ทุก PR
    deploy.yml       # deploy เมื่อ merge สู่ main
    security.yml     # security scan รายสัปดาห์
```

### 871.2 ส่วนประกอบของ Workflow

```yaml
name: ชื่อ workflow

on:                    # trigger event
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:                # job name
    runs-on: ubuntu-latest    # runner
    steps:
      - uses: actions/checkout@v4   # action
      - name: Run tests
        run: pytest                  # shell command
```

---

## ขั้นตอนที่ 872: CI Workflow — Test และ Lint

### 872.1 .github/workflows/ci.yml

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Cache pip
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements*.txt') }}

      - name: Install lint tools
        run: pip install ruff black isort

      - name: Run ruff
        run: ruff check apps/ config/

      - name: Check formatting (black)
        run: black --check apps/ config/

      - name: Check imports (isort)
        run: isort --check-only apps/ config/

  test:
    name: Test (Python ${{ matrix.python-version }})
    runs-on: ubuntu-latest
    needs: lint

    strategy:
      matrix:
        python-version: ['3.11', '3.12']

    services:
      postgres:
        image: postgres:15-alpine
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 5s
          --health-timeout 3s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Cache pip
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-${{ matrix.python-version }}-pip-${{ hashFiles('requirements*.txt') }}

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: Run tests with coverage
        env:
          DEBUG: "False"
          SECRET_KEY: test-secret-key-for-ci
          DATABASE_URL: postgres://test:test@localhost:5432/test_db
          REDIS_URL: redis://localhost:6379/0
          DJANGO_SETTINGS_MODULE: config.settings.testing
        run: |
          pytest --cov=apps --cov-report=xml --cov-report=term-missing -q

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          file: ./coverage.xml
          fail_ci_if_error: false
```

---

## ขั้นตอนที่ 873: Security Scan Workflow

### 873.1 .github/workflows/security.yml

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  push:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'  # ทุกวันจันทร์ เวลา 02:00 UTC

jobs:
  bandit:
    name: Bandit Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install bandit
      - run: bandit -r apps/ -ll -f json -o bandit-report.json || true
      - name: Upload bandit report
        uses: actions/upload-artifact@v4
        with:
          name: bandit-report
          path: bandit-report.json

  pip-audit:
    name: Dependency Audit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install pip-audit
      - run: pip-audit -r requirements.txt --desc
```

---

## ขั้นตอนที่ 874: Build Docker Image Workflow

### 874.1 .github/workflows/docker.yml

```yaml
# .github/workflows/docker.yml
name: Build and Push Docker Image

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=sha,prefix=sha-

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./Dockerfile.prod
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## ขั้นตอนที่ 875: Deploy Workflow

### 875.1 Deploy ไป VPS ผ่าน SSH

```yaml
# .github/workflows/deploy.yml
name: Deploy to Production

on:
  push:
    branches: [main]

jobs:
  test:
    uses: ./.github/workflows/ci.yml  # reuse CI workflow

  deploy:
    needs: test
    runs-on: ubuntu-latest
    environment: production  # ต้อง approve ใน GitHub settings

    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/myapp
            git pull origin main
            docker compose -f docker-compose.yml -f docker-compose.prod.yml pull
            docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d --no-deps web celery
            docker compose exec -T web python manage.py migrate --noinput
            docker compose exec -T web python manage.py collectstatic --noinput
            echo "Deploy completed at $(date)"
```

---

## ขั้นตอนที่ 876: GitHub Secrets Configuration

### 876.1 ตั้งค่า Secrets

ไปที่ **Settings > Secrets and variables > Actions** แล้วเพิ่ม:

| Secret Name | ค่า |
|-------------|-----|
| `SSH_HOST` | IP หรือ hostname ของ VPS |
| `SSH_USER` | username (เช่น `deploy`) |
| `SSH_PRIVATE_KEY` | private key สำหรับ SSH |
| `SECRET_KEY` | Django SECRET_KEY |
| `DATABASE_URL` | production database URL |
| `SENTRY_DSN` | Sentry DSN (ถ้าใช้) |

---

## ขั้นตอนที่ 877: Matrix Testing

### 877.1 ทดสอบหลาย Django Versions

```yaml
jobs:
  test:
    strategy:
      matrix:
        python-version: ['3.11', '3.12']
        django-version: ['4.2', '5.0']
        exclude:
          - python-version: '3.11'
            django-version: '5.0'  # Django 5.0 ต้องการ Python 3.10+

    steps:
      - name: Install specific Django version
        run: |
          pip install Django==${{ matrix.django-version }}.*
          pip install -r requirements.txt
```

---

## ขั้นตอนที่ 878: Workflow Reuse และ Composite Actions

### 878.1 Composite Action สำหรับ Setup

```yaml
# .github/actions/setup-django/action.yml
name: 'Setup Django'
description: 'Install Python, dependencies, and configure Django'
inputs:
  python-version:
    required: false
    default: '3.12'

runs:
  using: 'composite'
  steps:
    - uses: actions/setup-python@v5
      with:
        python-version: ${{ inputs.python-version }}

    - uses: actions/cache@v4
      with:
        path: ~/.cache/pip
        key: ${{ runner.os }}-pip-${{ hashFiles('requirements*.txt') }}

    - name: Install dependencies
      shell: bash
      run: pip install -r requirements.txt -r requirements-dev.txt
```

```yaml
# ใช้ composite action
steps:
  - uses: ./.github/actions/setup-django
    with:
      python-version: '3.12'
```

---

## ขั้นตอนที่ 879: Notifications

### 879.1 แจ้งเตือนผ่าน Slack

```yaml
- name: Notify Slack on success
  if: success()
  uses: slackapi/slack-github-action@v1
  with:
    payload: |
      {
        "text": "✅ Deploy สำเร็จ: ${{ github.repository }} @ ${{ github.sha }}"
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

- name: Notify Slack on failure
  if: failure()
  uses: slackapi/slack-github-action@v1
  with:
    payload: |
      {
        "text": "❌ Deploy ล้มเหลว: ${{ github.repository }} — ดู <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|workflow>"
      }
  env:
    SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## ขั้นตอนที่ 880: สรุปและแบบฝึกหัด

### 880.1 CI/CD Pipeline Architecture

```
Developer pushes code
         │
         ▼
   [GitHub Actions CI]
   ├── Lint (ruff, black)
   ├── Tests (pytest + coverage)
   └── Security scan (bandit, pip-audit)
         │
         ▼ (เฉพาะ main branch)
   [Build Docker Image]
   └── Push to GHCR
         │
         ▼ (with approval)
   [Deploy to Production]
   ├── Pull latest image
   ├── Run migrations
   └── Restart services
```

### 880.2 แบบฝึกหัด

**แบบฝึกหัดที่ 1**: สร้าง `.github/workflows/ci.yml` ที่รัน lint + test พร้อม services postgres และ redis ให้ผ่าน

**แบบฝึกหัดที่ 2**: เพิ่ม matrix testing ทดสอบ Python 3.11 และ 3.12

**แบบฝึกหัดที่ 3**: สร้าง deploy workflow ที่ deploy ผ่าน SSH เมื่อ push ไป main branch โดยผ่าน environment approval
