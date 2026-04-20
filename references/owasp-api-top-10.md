# OWASP API Security Top 10 (2023) — Reference

Focused on REST/GraphQL/gRPC APIs. Critical and High severity patterns.

---

## API1 — Broken Object Level Authorization (BOLA / IDOR)

**What it is:** API endpoints expose object identifiers; authorization check is missing, allowing any authenticated user to access any object.

**Detection:**
```
GET /api/v1/users/1234/profile       # Can user 5678 fetch this?
GET /api/v1/orders?user_id=1234      # Is user_id validated server-side?
DELETE /api/v1/documents/42          # Does the server check ownership?
```

**Severity:** BOLA affecting any user's private data → **Critical**.

**Mitigation:** Check that `resource.owner_id == authenticated_user.id` server-side on every operation. Use authorization middleware or ORM-level row-level security.

---

## API2 — Broken Authentication

**What it is:** Authentication mechanism is weak, bypassable, or not applied consistently to all endpoints.

**Detection:**
- API endpoints accessible without a valid token (forgotten `@require_auth` decorator)
- Tokens not validated (algorithm confusion in JWT: `alg: none`)
- Token not checked for expiry
- API keys in query parameters (logged in access logs, proxies)

**Code patterns:**
```python
# JWT algorithm confusion
jwt.decode(token, key, algorithms=["HS256", "RS256", "none"])  # never allow "none"

# API key in URL
GET /api/data?api_key=secret123  # exposed in server logs
```

**Severity:** Auth bypass on any endpoint → **Critical**.

---

## API3 — Broken Object Property Level Authorization

**What it is:** API allows a user to read or write object properties they shouldn't have access to (mass assignment, over-posting, excessive data exposure).

**Detection:**
```python
# Mass assignment: user can set is_admin by including it in the request body
user = User(**request.json)  # if request.json contains {"is_admin": true}
db.session.add(user)

# Excessive data exposure: internal fields returned
return jsonify(user.__dict__)  # includes password_hash, internal_id, etc.
```

**Severity:** Mass assignment that allows privilege escalation → **Critical**. Sensitive field exposure (password hash, SSN) → High.

**Mitigation:** Explicit allow-lists for request body fields (not `**kwargs`); explicit serializer that whitelists response fields.

---

## API4 — Unrestricted Resource Consumption

**What it is:** API has no limits on request rate, payload size, query complexity, or response size.

**Detection:**
- No rate limiting on authentication, registration, or password-reset endpoints
- No max payload size limit
- GraphQL queries with no depth/complexity limits
- LLM endpoints with no `max_tokens` and no per-user limits

**Severity:** Unrestricted auth endpoint (brute force) → High. Unrestricted LLM endpoint → **Critical** (financial and availability).

---

## API5 — Broken Function Level Authorization

**What it is:** Administrative or privileged API functions are accessible to unprivileged users.

**Detection:**
```
POST /api/v1/admin/users          # is this checked server-side?
DELETE /api/v1/users/{id}         # regular users should only delete themselves
PUT /api/v1/users/{id}/role       # role assignment should be admin-only
```

**Severity:** Regular user can call admin function → **Critical**.

---

## API6 — Unrestricted Access to Sensitive Business Flows

**What it is:** API endpoints supporting sensitive business flows (purchases, transfers, bookings) can be abused by automated scripts at scale.

**Detection:**
- No CAPTCHA or behavioral analysis on checkout / purchase flows
- No per-account rate limit on bookings, coupon redemption, referral bonuses
- Coupon codes that can be brute-forced

**Severity:** Business logic that can be financially exploited at scale → High.

---

## API7 — Server Side Request Forgery

Same as OWASP Web A10 — see `owasp-top-10-web.md`. 

**Severity:** SSRF reaching cloud metadata endpoints → **Critical**.

---

## API8 — Security Misconfiguration

Same pattern as OWASP Web A05 applied to API layer:
- CORS too permissive (`*` with credentials)
- Default error responses exposing framework details
- HTTP methods not restricted (`TRACE`, `OPTIONS` returning sensitive headers)
- Missing TLS on internal API communication

**Severity:** Permissive CORS with credentials → High.

---

## API9 — Improper Inventory Management

**What it is:** Outdated or shadow API versions remain accessible and don't receive security patches.

**Detection:**
- `/api/v1/` endpoints still live when `/api/v3/` is the supported version
- Debug or test endpoints accessible in production (`/api/debug`, `/api/test`)
- Undocumented internal endpoints reachable from outside

**Severity:** Debug endpoint in production → High. Outdated API version with known vulnerability → map to that vulnerability's severity.

---

## API10 — Unsafe Consumption of APIs

**What it is:** The application blindly trusts and uses data from third-party APIs without validation, leading to injection or data integrity issues.

**Detection:**
```python
# Data from third-party API used directly in SQL query
third_party_data = external_api.get_user_profile(user_id)
query = f"INSERT INTO profiles VALUES ('{third_party_data['name']}')"  # SQL injection

# Third-party API response rendered directly in UI
div.innerHTML = fetch_from_partner_api();  # XSS
```

**Severity:** Third-party data used in SQL/HTML/shell without sanitization → **Critical** (maps to injection).

**Mitigation:** Treat third-party API responses as untrusted external data; validate and sanitize before use.
