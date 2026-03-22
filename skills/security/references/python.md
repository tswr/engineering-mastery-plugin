# Python Security Reference

Python-specific vulnerability patterns and mitigations. Python's dynamic nature and batteries-included standard library introduce unique security risks around deserialization, command execution, and input handling.

---

## SQL Injection

```python
# DANGEROUS: string formatting in queries
def find_user(username: str) -> User:
    query = f"SELECT * FROM users WHERE name = '{username}'"
    cursor.execute(query)  # username = "'; DROP TABLE users; --"

# SAFE: parameterized queries (works with any DB driver)
def find_user(username: str) -> User:
    cursor.execute("SELECT * FROM users WHERE name = %s", (username,))

# SAFE: ORM (SQLAlchemy, Django ORM) — parameterized by default
user = session.query(User).filter(User.name == username).first()

# STILL DANGEROUS: raw SQL through ORM
session.execute(text(f"SELECT * FROM users WHERE name = '{username}'"))

# SAFE: parameterized raw SQL through ORM
session.execute(text("SELECT * FROM users WHERE name = :name"), {"name": username})
```

## Command Injection

```python
import subprocess
import shlex

# DANGEROUS: shell=True with user input
def convert_file(filename: str) -> None:
    subprocess.run(f"convert {filename} output.png", shell=True)
    # filename = "; rm -rf /" → catastrophe

# DANGEROUS: os.system always invokes a shell
os.system(f"convert {filename} output.png")

# SAFE: argument list without shell
def convert_file(filename: str) -> None:
    subprocess.run(["convert", filename, "output.png"], check=True)
    # filename is passed as a single argument, never shell-interpreted

# If you genuinely need shell features (pipes, redirection):
# Use subprocess.PIPE and chain processes manually
p1 = subprocess.Popen(["grep", pattern, filename], stdout=subprocess.PIPE)
p2 = subprocess.Popen(["wc", "-l"], stdin=p1.stdout, stdout=subprocess.PIPE)
output = p2.communicate()[0]
```

## Deserialization

```python
# DANGEROUS: pickle executes arbitrary code during deserialization
import pickle

def load_data(raw_bytes: bytes):
    return pickle.loads(raw_bytes)  # attacker-controlled bytes → code execution

# pickle, shelve, marshal are ALL unsafe for untrusted data
# There is no way to make pickle safe against malicious input

# SAFE: use data-only formats
import json
def load_data(raw_bytes: bytes):
    return json.loads(raw_bytes)  # only parses data, no code execution

# SAFE: YAML with safe loader
import yaml
data = yaml.safe_load(raw_string)  # NOT yaml.load() which is unsafe by default

# DANGEROUS: yaml.load without SafeLoader
data = yaml.load(raw_string)  # can execute arbitrary Python


# For complex serialization needs:
# - msgpack, protobuf, flatbuffers for binary formats
# - Pydantic or dataclasses-json for typed JSON parsing
from pydantic import BaseModel

class UserData(BaseModel):
    name: str
    age: int

user = UserData.model_validate_json(raw_bytes)  # typed, validated, safe
```

## Path Traversal

```python
from pathlib import Path

# DANGEROUS: user controls file path
def serve_file(user_path: str) -> bytes:
    with open(f"/data/uploads/{user_path}", "rb") as f:
        return f.read()
    # user_path = "../../etc/passwd" → reads arbitrary files

# SAFE: resolve and verify
def serve_file(user_path: str) -> bytes:
    base = Path("/data/uploads").resolve()
    full = (base / user_path).resolve()

    if not full.is_relative_to(base):
        raise ValueError("Path traversal attempt")

    with open(full, "rb") as f:
        return f.read()

# Also watch for null bytes in older Python versions
# and symbolic links that escape the directory
```

## Secrets and Credentials

```python
# DANGEROUS: hardcoded secrets
API_KEY = "sk-1234567890abcdef"  # in source code, in version control

# SAFE: environment variables
import os
API_KEY = os.environ["API_KEY"]  # set externally, not in code

# BETTER: secret management service
from cloud_provider import SecretManager
API_KEY = SecretManager.get_secret("api-key")


# DANGEROUS: predictable random for security tokens
import random
token = random.randint(0, 2**64)  # Mersenne Twister — predictable

# SAFE: cryptographically secure random
import secrets
token = secrets.token_urlsafe(32)  # 256 bits of entropy, URL-safe encoding
session_id = secrets.token_hex(32)  # 256 bits as hex string


# SAFE: constant-time comparison for secrets
import hmac
def verify_token(provided: str, expected: str) -> bool:
    return hmac.compare_digest(provided, expected)  # constant-time

# DANGEROUS: regular comparison leaks timing information
if provided_token == stored_token:  # early exit reveals prefix length
    ...
```

## Template Injection

```python
# DANGEROUS: user input in format strings
def greet(user_input: str) -> str:
    template = f"Hello, {user_input}!"  # f-strings are safe for simple values
    return template

# DANGEROUS: user input AS the template
def render(user_template: str, **kwargs) -> str:
    return user_template.format(**kwargs)
    # user_template = "{0.__class__.__mro__[1].__subclasses__()}"
    # → can access arbitrary Python objects

# DANGEROUS: user input in Jinja2 without sandboxing
from jinja2 import Template
Template(user_input).render()  # arbitrary code execution

# SAFE: use Jinja2 sandboxed environment and render user data as values
from jinja2 import Environment, select_autoescape
env = Environment(autoescape=select_autoescape())
template = env.get_template("greeting.html")  # template from trusted source
output = template.render(name=user_input)       # user data as a value, not template


# DANGEROUS: eval/exec with user input — never do this
result = eval(user_expression)    # arbitrary code execution
exec(user_code)                   # arbitrary code execution
```

## Web Security (Flask/Django)

```python
# XSS: Django and Jinja2 auto-escape HTML by default
# DANGEROUS: marking user input as safe
from django.utils.safestring import mark_safe
mark_safe(user_input)  # disables escaping — XSS if input contains <script>
from jinja2 import Markup
Markup(user_input)      # same problem

# SAFE: let the template engine escape (which it does by default)
# {{ user_input }} in Django/Jinja2 is auto-escaped


# CSRF: Django includes CSRF protection by default
# Never disable it: @csrf_exempt  # only for APIs with token-based auth


# SSRF: validating URLs from user input
# DANGEROUS: fetching a user-provided URL without validation
import requests
response = requests.get(user_url)  # user_url = "http://169.254.169.254/..."
                                    # → accesses cloud metadata service

# SAFE: validate the URL against an allowlist of hosts/schemes
from urllib.parse import urlparse
parsed = urlparse(user_url)
ALLOWED_HOSTS = {"api.example.com", "cdn.example.com"}
if parsed.hostname not in ALLOWED_HOSTS:
    raise ValueError("URL not allowed")
if parsed.scheme not in ("https",):
    raise ValueError("HTTPS required")
# Also block private IP ranges: 10.x, 172.16-31.x, 192.168.x, 169.254.x, 127.x


# HTTP headers
# Set security headers in responses:
# Strict-Transport-Security: max-age=31536000; includeSubDomains
# Content-Security-Policy: default-src 'self'
# X-Content-Type-Options: nosniff
# X-Frame-Options: DENY
```

## Dependency Auditing

```bash
# pip-audit: check for known vulnerabilities
pip install pip-audit
pip-audit

# safety: another vulnerability scanner
pip install safety
safety check

# In CI: fail the build on known vulnerabilities
pip-audit --strict

# Pin dependencies with hashes for supply chain integrity
pip install --require-hashes -r requirements.txt

# Generate requirements with hashes:
pip-compile --generate-hashes requirements.in
```
