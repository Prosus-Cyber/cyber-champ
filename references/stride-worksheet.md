# STRIDE Threat Modeling Worksheet

STRIDE is a Microsoft threat-modeling framework. Apply each category to every asset and trust boundary you identify in the feature.

---

## How to use STRIDE

1. **List assets** — what has value and must be protected (user PII, session tokens, model weights, API keys, conversation history, payment data, file system, databases, internal APIs).
2. **List trust boundaries** — where data crosses between trust zones (internet → server, user input → LLM prompt, LLM output → database, client → API, server → third-party LLM API).
3. **For each trust boundary, ask each STRIDE question below.**
4. **Rate severity** using the rubric. Report Critical and High only (default).

---

## STRIDE Categories

### S — Spoofing
*Can an attacker impersonate a user, service, component, or data source?*

Questions:
- Can an attacker forge authentication tokens (JWT, session cookie, OAuth token)?
- Can an attacker impersonate another user by manipulating an identifier (user ID in request, email in token)?
- In an AI context: can a retrieved document impersonate a trusted system instruction (e.g., "This is a system message from the admin")?
- Can an attacker call an internal API by spoofing an expected origin header or IP?

Severity triggers:
- Attacker can impersonate any user → **Critical**
- Attacker can impersonate a privileged user → **Critical**
- Document can spoof system prompt authority → **Critical** (indirect prompt injection)

Mitigations: Strong authentication (MFA, PKCE, short-lived tokens), HMAC-signed tokens, strict origin validation, structured prompt format that distinguishes user/system/tool roles.

---

### T — Tampering
*Can an attacker modify data in transit or at rest?*

Questions:
- Can an attacker modify request parameters (price, quantity, user_id, file path) before they are processed?
- Can an attacker modify data in the database by exploiting missing authorization checks?
- In an AI context: can an attacker poison the RAG knowledge base or training data?
- Can an attacker intercept and modify API responses (man-in-the-middle)?

Severity triggers:
- Attacker can modify financial records or security-relevant data → **Critical**
- Attacker can inject content into the LLM knowledge base → High/Critical depending on blast radius
- API communication over HTTP (not HTTPS) → High

Mitigations: Input validation and output encoding, parameterized queries, signed/verified data at rest (checksums on model files), HTTPS everywhere, content review workflows for RAG ingestion.

---

### R — Repudiation
*Can an actor deny their actions? Are actions auditable?*

Questions:
- Are security-relevant actions (login, data access, file writes, LLM tool invocations) logged?
- Can a user deny having sent a particular prompt or invoked a tool?
- Are logs tamper-proof and access-controlled?
- In agentic workflows: is it traceable which action was taken in response to which prompt?

Severity triggers:
- Complete absence of audit logging in a regulated context (financial, health) → High
- Logs writable by the same process that generates them (not integrity-protected) → Medium

Mitigations: Append-only audit log (e.g., CloudTrail, Datadog audit logs), log user ID + action + timestamp + input/output hash, ensure logs are in a separate security boundary from the application.

---

### I — Information Disclosure
*Can an attacker read data they are not authorized to access?*

Questions:
- Can an API endpoint return data belonging to another user by changing an ID in the request?
- Can an attacker extract PII or secrets from error messages, logs, or verbose API responses?
- In an AI context: can a user craft a prompt that causes the LLM to repeat back another user's conversation, PII, or the system prompt?
- Can an attacker retrieve sensitive documents from the RAG knowledge base that they should not access?
- Are credentials, API keys, or secrets hardcoded in the codebase or configuration?

Severity triggers:
- IDOR exposing another user's PII → **Critical**
- System prompt contains credentials → **Critical** (LLM07 + I)
- Stack traces / SQL errors exposed in production API → Medium/High

Mitigations: Object-level authorization checks on every data access, per-user namespace filtering on vector store queries, output filtering before returning LLM responses, secrets in environment variables / secret manager (not in code).

---

### D — Denial of Service
*Can an attacker prevent legitimate users from accessing the service?*

Questions:
- Can an unauthenticated attacker trigger expensive LLM API calls, exhausting quota or budget?
- Can an attacker craft a large input that causes the server to allocate excessive memory or CPU?
- Can an attacker flood an API endpoint without rate limiting?
- In an agentic context: can a prompt cause an agent to loop indefinitely or spawn excessive sub-agents?

Severity triggers:
- Unauthenticated LLM API endpoint with no rate limiting or `max_tokens` → **Critical**
- Authenticated endpoint missing rate limits → High
- Agent without a maximum iteration or timeout → High

Mitigations: Rate limiting per user/IP, `max_tokens` on LLM calls, input length validation, agent iteration caps, circuit breakers.

---

### E — Elevation of Privilege
*Can an attacker gain capabilities or access they should not have?*

Questions:
- Can a regular user access admin functionality by manipulating a parameter or header?
- Can a user access another user's resources by changing an ID (IDOR — also I, but privilege dimension here)?
- In an AI context: can a user craft a prompt that instructs the LLM agent to use tools beyond their authorization level?
- Can the LLM be instructed to act as an admin or to bypass application-level access controls?
- Can an attacker use a compromised LLM response to inject code or commands that execute with elevated privileges on the server?

Severity triggers:
- User → admin via prompt injection (agent with admin tools) → **Critical**
- LLM agent performing operations on behalf of any user, not the authenticated user → **Critical**
- Missing role-based authorization on API endpoints → High

Mitigations: Server-side authorization checks (do not rely on client-side or LLM-enforced restrictions), scope agent tool permissions to the authenticated user's own resources, ensure the agent acts as the user (not as a super-user), validate role claims server-side.

---

## STRIDE × AI Quick Reference

| STRIDE | Classic example | AI-specific twist |
|---|---|---|
| Spoofing | Forged JWT | Retrieved doc impersonates system message |
| Tampering | SQL injection | RAG knowledge base poisoning |
| Repudiation | No access logs | No trace of which prompt triggered which agent action |
| Info Disclosure | IDOR exposes PII | Prompt extraction of system prompt or another user's context |
| Denial of Service | API flood | Prompt that triggers expensive/looping agent behaviour |
| Elevation of Privilege | IDOR → admin | Prompt injection that causes agent to use admin-scoped tools |
