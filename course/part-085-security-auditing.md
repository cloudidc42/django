# Part 085: Security Auditing และ Penetration Testing เบื้องต้น

> **ขั้นตอนที่ 841-850** | Phase 10: Security

---

## ขั้นตอนที่ 841: Django Security Check

### 841.1 `manage.py check --deploy`

```bash
# ตรวจสอบ security configuration สำหรับ production
python manage.py check --deploy

# ตัวอย่าง output ที่ดี:
# System check identified no issues (0 silenced).

# ตัวอย่าง output ที่มีปัญหา:
# WARNINGS:
# ?: (security.W004) You have not set a value for the SECURE_HSTS_SECONDS setting.
# ?: (security.W008) Your SECRET_KEY has less than 50 characters.
# ?: (security.W012) SESSION_COOKIE_SECURE is not set to True.
```

### 841.2 รัน Check ใน CI/CD

```yaml
# .github/workflows/security.yml
jobs:
  security-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Django security check
        run: |
          python manage.py check --deploy --fail-level WARNING
        env:
          DJANGO_SETTINGS_MODULE: config.settings.production
          SECRET_KEY: ${{ secrets.SECRET_KEY }}
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
```

---

## ขั้นตอนที่ 842: Bandit — Static Code Analysis

### 842.1 ติดตั้งและใช้งาน

```bash
pip install bandit

# scan โปรเจกต์ทั้งหมด
bandit -r apps/ -ll

# scan และ export เป็น JSON
bandit -r apps/ -f json -o bandit-report.json

# ข้าม test files
bandit -r apps/ --exclude "*/tests/*,*/test_*"
```

### 842.2 ตีความผล Bandit

```
Code  Issue
B101  assert_used — ใช้ assert ใน production code
B102  exec_used — ใช้ exec()
B103  set_bad_file_permissions — permissions ไม่ปลอดภัย
B108  hardcoded_tmp_directory — path /tmp hardcoded
B301  pickle — ใช้ pickle deserialize untrusted data
B323  unverified_context — ssl.create_unverified_context
B501  request_with_no_cert_validation — verify=False
B602  subprocess_popen_with_shell_equals_true
B608  hardcoded_sql_expressions — SQL injection risk
```

```python
# ถ้า false positive ใช้ comment ปิด
result = subprocess.run(cmd, shell=True)  # nosec B602

# หรือปิดทั้ง line
assert condition  # nosec
```

---

## ขั้นตอนที่ 843: pip-audit และ Safety

### 843.1 ตรวจสอบ Dependencies มี CVE หรือไม่

```bash
# pip-audit (แนะนำ — ใช้ OSV database)
pip install pip-audit
pip-audit
pip-audit --fix  # auto-upgrade packages ที่มี vuln

# safety (ใช้ Safety DB)
pip install safety
safety check
safety check --json

# ตรวจสอบ requirements.txt
safety check -r requirements.txt
```

### 843.2 Integrate กับ CI

```yaml
# .github/workflows/security.yml
jobs:
  dependency-audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Install dependencies
        run: pip install -r requirements.txt pip-audit
      - name: Audit dependencies
        run: pip-audit --fail-on-cvss 7.0
```

---

## ขั้นตอนที่ 844: Django Security Middleware Audit

### 844.1 ตรวจสอบ Middleware Configuration

```python
# scripts/audit_middleware.py
import django
import os
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'config.settings.production')
django.setup()

from django.conf import settings

REQUIRED_MIDDLEWARE = {
    'django.middleware.security.SecurityMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
}

DANGEROUS_MIDDLEWARE_PATTERNS = [
    'CorsMiddleware',  # ต้องตรวจสอบ CORS_ALLOWED_ORIGINS
]

current = set(settings.MIDDLEWARE)
missing = REQUIRED_MIDDLEWARE - current

if missing:
    print(f"MISSING middleware: {missing}")
else:
    print("OK: All required middleware present")

for m in settings.MIDDLEWARE:
    for pattern in DANGEROUS_MIDDLEWARE_PATTERNS:
        if pattern in m:
            print(f"REVIEW: {m} — ตรวจสอบ configuration")
```

---

## ขั้นตอนที่ 845: SQL Injection Audit

### 845.1 ค้นหา Raw SQL ในโค้ด

```bash
# ค้นหาการใช้ raw SQL
grep -rn "raw\|execute\|RawSQL" apps/ --include="*.py" | grep -v test | grep -v ".pyc"

# ค้นหา string interpolation ใน SQL
grep -rn "f\"SELECT\|f'SELECT\|\%s.*FROM" apps/ --include="*.py"
```

### 845.2 Pattern ที่ปลอดภัย vs อันตราย

```python
# ❌ อันตราย — SQL Injection
def get_user(username):
    from django.db import connection
    with connection.cursor() as cursor:
        cursor.execute(f"SELECT * FROM auth_user WHERE username = '{username}'")

# ✅ ปลอดภัย — Parameterized query
def get_user_safe(username):
    from django.db import connection
    with connection.cursor() as cursor:
        cursor.execute("SELECT * FROM auth_user WHERE username = %s", [username])

# ✅ ดีที่สุด — ใช้ ORM
def get_user_orm(username):
    from django.contrib.auth import get_user_model
    return get_user_model().objects.filter(username=username).first()
```

---

## ขั้นตอนที่ 846: XSS Audit

### 846.1 ค้นหา `mark_safe` และ `safe` filter

```bash
# ค้นหาการใช้ mark_safe ใน views/models
grep -rn "mark_safe\|format_html" apps/ --include="*.py"

# ค้นหา |safe filter ใน templates
grep -rn "| safe\||safe" templates/
```

### 846.2 ตรวจสอบ Template Context

```python
# tests/test_xss.py
import pytest
from django.test import Client


@pytest.mark.django_db
class TestXSSPrevention:
    def setup_method(self):
        self.client = Client()

    def test_user_input_is_escaped(self, user_factory):
        """ตรวจสอบว่า user input ถูก escape"""
        user = user_factory(username='<script>alert(1)</script>')
        self.client.force_login(user)
        response = self.client.get('/profile/')
        assert '<script>alert(1)</script>' not in response.content.decode()
        assert '&lt;script&gt;' in response.content.decode()

    def test_search_query_is_escaped(self):
        response = self.client.get('/search/?q=<script>alert(1)</script>')
        assert '<script>' not in response.content.decode()
```

---

## ขั้นตอนที่ 847: Security Headers Audit

### 847.1 Automated Header Check

```python
# scripts/check_security_headers.py
import urllib.request
import ssl

REQUIRED_HEADERS = {
    'Strict-Transport-Security': lambda v: 'max-age=' in v,
    'X-Frame-Options': lambda v: v in ('DENY', 'SAMEORIGIN'),
    'X-Content-Type-Options': lambda v: v == 'nosniff',
    'Content-Security-Policy': lambda v: len(v) > 0,
    'Referrer-Policy': lambda v: v in (
        'no-referrer', 'strict-origin', 'strict-origin-when-cross-origin'
    ),
}


def audit_headers(url: str):
    ctx = ssl.create_default_context()
    req = urllib.request.Request(url)
    with urllib.request.urlopen(req, context=ctx) as response:
        headers = dict(response.headers)

    issues = []
    for header, validator in REQUIRED_HEADERS.items():
        value = headers.get(header, '')
        if not value:
            issues.append(f"MISSING: {header}")
        elif not validator(value):
            issues.append(f"WEAK: {header} = {value}")
        else:
            print(f"OK: {header} = {value}")

    for issue in issues:
        print(issue)


if __name__ == '__main__':
    audit_headers('https://example.com')
```

---

## ขั้นตอนที่ 848: Dependency Confusion Attack Prevention

### 848.1 ป้องกัน Supply Chain Attack

```bash
# ใช้ hash verification ใน requirements
pip install --require-hashes -r requirements.txt

# สร้าง requirements พร้อม hashes
pip-compile --generate-hashes requirements.in
```

```
# requirements.txt (ตัวอย่างพร้อม hashes)
Django==4.2.7 \
    --hash=sha256:b8f2af5bb6... \
    --hash=sha256:d7c0c8...
```

```python
# pyproject.toml — pin exact versions
[tool.poetry.dependencies]
python = "^3.11"
Django = "4.2.7"  # exact pin ใน production
```

---

## ขั้นตอนที่ 849: Penetration Testing เบื้องต้น

### 849.1 OWASP ZAP Baseline Scan

```bash
# รัน ZAP baseline scan ด้วย Docker
docker run -t owasp/zap2docker-stable zap-baseline.py \
  -t https://staging.example.com \
  -r zap-report.html

# Full scan (นานกว่า)
docker run -t owasp/zap2docker-stable zap-full-scan.py \
  -t https://staging.example.com \
  -r zap-full-report.html
```

### 849.2 Manual Testing Checklist

**Authentication:**
- [ ] ทดสอบ login ด้วย credentials ผิด หลาย ๆ ครั้ง → ต้อง lock
- [ ] ทดสอบ password reset token ใช้ซ้ำได้หรือเปล่า
- [ ] ทดสอบ session ยัง valid หลัง logout หรือเปล่า

**Authorization:**
- [ ] ทดสอบ access object ของ user อื่น (IDOR)
- [ ] ทดสอบ admin endpoint โดยไม่มีสิทธิ์

**Input Validation:**
- [ ] ทดสอบ SQL injection บน search fields
- [ ] ทดสอบ XSS บน input fields ทุกอัน
- [ ] ทดสอบ file upload ด้วยไฟล์ชนิดอื่น

---

## ขั้นตอนที่ 850: สรุปและ Security Audit Workflow

### 850.1 Security Audit Automation Script

```bash
#!/bin/bash
# scripts/security_audit.sh

echo "=== Django Security Audit ==="

echo "\n--- 1. Django deployment check ---"
python manage.py check --deploy 2>&1

echo "\n--- 2. Bandit static analysis ---"
bandit -r apps/ -ll -q 2>&1 | tail -20

echo "\n--- 3. Dependency vulnerabilities ---"
pip-audit --desc 2>&1

echo "\n--- 4. Check for hardcoded secrets ---"
grep -rn "password.*=.*['\"].[์{8,}]['\"]" apps/ --include="*.py" | grep -v test | grep -v "placeholder\|example\|changeme" 2>&1

echo "\n=== Audit Complete ==="
```

### 850.2 แบบฝึกหัด

**แบบฝึกหัดที่ 1**: รัน `bandit -r apps/` บนโปรเจกต์ของคุณ แก้ไข HIGH severity ทุกตัว

**แบบฝึกหัดที่ 2**: รัน `pip-audit` และ upgrade packages ที่มี CVE severity ≥ 7.0

**แบบฝึกหัดที่ 3**: เพิ่ม security audit ใน GitHub Actions ที่ fail เมื่อ bandit พบ HIGH severity หรือ pip-audit พบ CVSS ≥ 7
