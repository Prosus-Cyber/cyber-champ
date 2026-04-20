# OWASP Top 10 Web (2021) — Security Reference

Concise detection patterns and mitigations for each category. Focus on Critical and High severity instances.

---

## A01 — Broken Access Control

**What to look for:**
- API endpoints that don't verify the authenticated user owns the resource being accessed (IDOR)
- Admin routes missing server-side role check (`if user.is_admin` only in the UI, not backend)
- JWT / session claims trusted on the client side
- Forced browsing: `/admin`, `/api/users/all`, `/internal` reachable without auth
- CORS misconfiguration allowing arbitrary origins with credentials

**Code patterns:**
```python
# Missing authorization check — IDOR
@app.get("/api/documents/{doc_id}")
def get_document(doc_id: int, db: Session = Depends(get_db)):
    return db.query(Document).filter(Document.id == doc_id).first()
    # Missing: .filter(Document.owner_id == current_user.id)

# Role check only on frontend
if (user.role === 'admin') { showAdminPanel(); }
# Backend endpoint has no equivalent check
```

**Severity:** Missing ownership check on user data → **Critical** (mass IDOR). Missing role check on admin endpoint → **Critical**.

**Mitigations:** Enforce authorization server-side on every request; default-deny policy; object-level authorization checks before every data access.

---

## A02 — Cryptographic Failures

**What to look for:**
- PII or sensitive data transmitted over HTTP (not HTTPS)
- Passwords stored in plaintext or with a fast hash (MD5, SHA1, SHA256 without salt)
- Hardcoded secrets or API keys in source code
- Weak or deprecated algorithms (MD5, SHA1, DES, ECB mode)
- Insufficient randomness for security-sensitive values (session IDs, tokens, OTPs)

**Code patterns:**
```python
# Weak password hash
password_hash = hashlib.md5(password.encode()).hexdigest()

# Hardcoded secret
API_KEY = "sk-live-abc123..."

# Weak random for session token
import random
session_id = str(random.randint(0, 999999))

# ECB mode
cipher = AES.new(key, AES.MODE_ECB)
```

**Severity:** Hardcoded production credentials → **Critical**. Passwords stored in plaintext or MD5 → **Critical**. HTTP for authenticated sessions → High.

**Mitigations:** bcrypt/argon2 for passwords; secrets in environment variables / vault; `secrets.token_urlsafe()` for tokens; TLS everywhere; AES-GCM or ChaCha20-Poly1305.

---

## A03 — Injection (SQL, Command, Template, LDAP, XPath)

**What to look for:**
- User input concatenated into SQL queries
- User input passed to `os.system`, `subprocess`, `exec`, `eval`
- User input rendered in templates without escaping
- User input used in LDAP/XPath queries

**Code patterns:**
```python
# SQL injection
query = f"SELECT * FROM users WHERE email = '{user_email}'"
db.execute(query)

# Command injection
os.system(f"convert {filename} output.pdf")

# Server-side template injection
template = Template(user_input)
template.render()

# XSS (stored)
document.innerHTML = user_supplied_content;
```

**Severity:** SQL injection with data read access → High. SQL injection with data write or auth bypass → **Critical**. Command injection → **Critical**.

**Mitigations:** Parameterized queries for SQL; `subprocess` with list args (not shell=True); template auto-escaping; `textContent` instead of `innerHTML`.

---

## A04 — Insecure Design

**What to look for (design-time):**
- No threat model exists for the feature
- Security relies solely on UI-level controls, not server-side enforcement
- Business logic that can be abused (price manipulation, coupon stacking, rate limit bypass by creating multiple accounts)
- Missing rate limiting on resource-intensive or sensitive operations
- Forgot-password flow with weak or predictable reset tokens

**Severity:** Business logic that allows financial manipulation → **Critical**. Missing rate limits on account operations (password reset, login) → High.

---

## A05 — Security Misconfiguration

**What to look for:**
- Default credentials left in place (`admin/admin`, `postgres/postgres`)
- Debug mode enabled in production (stack traces exposed, profiler endpoints open)
- Directory listing enabled on web server
- Unnecessary HTTP methods enabled (TRACE, PUT on a read-only API)
- Missing security headers (CSP, X-Frame-Options, X-Content-Type-Options)
- Overly permissive CORS (`Access-Control-Allow-Origin: *` with `allowCredentials: true`)

**Severity:** Default credentials in production → **Critical**. Debug/profiler endpoint exposed in production → High.

---

## A06 — Vulnerable and Outdated Components

**What to look for:**
- Dependencies with known CVEs (use `npm audit`, `pip audit`, `trivy`, Dependabot)
- Dependencies pinned to old major versions with critical vulnerabilities
- Transitive dependencies with unpatched vulnerabilities

**Severity:** Dependency with a Critical or High CVE exploitable in this application's context → map to the CVE's own severity.

---

## A07 — Identification and Authentication Failures

**What to look for:**
- No brute-force protection on login (missing rate limit or lockout)
- Weak session token (short, guessable, not invalidated on logout)
- Session fixation (session ID not rotated after login)
- Password complexity not enforced
- Missing MFA on sensitive operations (admin panel, payment, account deletion)

**Code patterns:**
```python
# Session not invalidated on logout
@app.post("/logout")
def logout():
    # Missing: session.clear() or session.invalidate()
    return redirect("/")
```

**Severity:** No brute-force protection on login → High. Session not invalidated on logout → High.

---

## A08 — Software and Data Integrity Failures

**What to look for:**
- CI/CD pipeline plugins or build tools from unverified sources
- Auto-update mechanisms that fetch and execute code without signature verification
- Deserialization of untrusted data without type constraints
- Object/prototype pollution in JavaScript deserialization

**Code patterns:**
```python
# Unsafe deserialization
import pickle
obj = pickle.loads(user_input)  # arbitrary code execution

# Prototype pollution
const merged = Object.assign({}, userInput);
```

**Severity:** Deserialization of untrusted data with `pickle`, `marshal`, or Java `ObjectInputStream` → **Critical**.

---

## A09 — Security Logging and Monitoring Failures

**What to look for:**
- Login failures, authorization failures, and high-value actions not logged
- Log injection: user input written to logs without sanitization
- No alerting on anomalous access patterns
- Logs contain full passwords or tokens (over-logging)

**Severity:** Absence of audit logging in a regulated context (finance, health) → High. Log injection enabling log forgery → Medium.

---

## A10 — Server-Side Request Forgery (SSRF)

**What to look for:**
- API endpoint that fetches a URL supplied by the user
- Image/file/webhook import features that make server-side HTTP requests
- LLM tools or integrations that fetch user-supplied URLs

**Code patterns:**
```python
# SSRF
@app.post("/preview")
def preview(url: str):
    return requests.get(url).content  # can fetch http://169.254.169.254/metadata
```

**Severity:** SSRF that can reach cloud metadata endpoints (AWS IMDSv1, GCP metadata) → **Critical**. SSRF blocked from external but can reach internal services → High.

**Mitigations:** Allowlist of permitted domains/schemes; block `169.254.169.254` and RFC 1918 ranges; use a web proxy with egress filtering.
