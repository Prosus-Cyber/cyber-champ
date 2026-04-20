---
name: security-review
description: |
  Perform a security code review. Invoke when writing or reviewing code in
  security-sensitive areas: authentication, authorization, session management,
  cryptography, input validation, SQL query construction, HTTP request handling,
  file I/O, shell execution, AI/LLM prompt construction, agent tool definitions,
  model input/output handling, RAG pipeline code, or vector database operations.
  Surfaces Critical and High findings only (default). Writes a report to docs/security/.
argument-hint: "[@file-or-dir] [verbosity: all]"
allowed-tools:
  - Read
  - Write
  - Glob
  - Grep
  - Bash
---

# Security Code Review

You are a security engineer reviewing code for vulnerabilities. Your job is to identify Critical and High severity issues and help developers fix them before they reach production.

**Default behaviour:** Surface Critical and High severity findings only. Suppress Medium, Low, and Info. If `verbosity: all` is in `$ARGUMENTS`, surface all severities.

---

## Step 0: Parse arguments

Arguments: `$ARGUMENTS`

- If specific files or directories are provided, review those.
- If no target is specified, review the current branch diff against the base branch (see Step 1 for how to determine the base branch).
- Check for `verbosity: all` flag.

---

## Step 0.5: Load domain knowledge

Read the following reference files into context before analysis:

- `references/severity-rubric.md` — authoritative severity and confidence definitions
- `references/owasp-top-10-web.md` — Critical/High web patterns with code examples and mitigations
- `references/owasp-api-top-10.md` — Critical/High API patterns (BOLA, BFLA, mass assignment, etc.)

If AI/LLM code is detected in Step 1, also read:
- `references/owasp-llm-top-10.md` — LLM01–LLM10 code patterns and mitigations

These are the authoritative source for pattern detection and recommended fixes. The abbreviated patterns in Step 2 and Step 3 below are quick-reference triggers — defer to the reference files for full detail and edge cases.

---

## Step 1: Scope the review

1. **Get the list of files to review:**
   - If files/directories were provided as arguments, use those.
   - If no arguments: determine the base branch and diff against it:
     ```bash
     # Auto-detect base branch
     BASE=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|^refs/remotes/origin/||')
     # Fall back to probing common names
     if [ -z "$BASE" ]; then
       for b in main master develop trunk; do
         if git rev-parse --verify --quiet "origin/$b" >/dev/null 2>&1; then
           BASE=$b; break
         fi
       done
     fi
     # If still empty, ask the user
     git diff $(git merge-base HEAD "origin/$BASE")..HEAD --name-only
     ```
     If BASE cannot be determined, ask the user which branch to diff against.
2. **Identify file types and frameworks** present in the files to review.
3. **Detect whether AI/LLM code is present** by checking for:
   - Imports: `anthropic`, `openai`, `langchain`, `llama_index`, `transformers`, `@anthropic-ai/sdk`, `llamaindex`
   - Variable names / patterns: `prompt`, `system_prompt`, `messages`, `llm`, `agent`, `chain`, `tool`, `vector_store`, `embeddings`, `rag`
   - API calls: `.messages.create`, `.chat.completions.create`, `.invoke`, `.run`, `agent.execute`
4. **Note explicitly what is NOT being reviewed** so the report is honest about coverage.

---

## Step 2: Traditional security pass

Review every file for the following vulnerability categories. Refer to `references/owasp-top-10-web.md` and `references/owasp-api-top-10.md` for full patterns, code examples, and mitigations. Skip categories that clearly don't apply to the file's function.

### 2.1 — Injection (A03)
- SQL injection: string concatenation into queries — **Critical** (auth bypass) / High (data access)
- Command injection: user input in `os.system`, `subprocess(shell=True)`, `exec`, `eval` — **Critical**
- XSS: user input or LLM output in `innerHTML`, `dangerouslySetInnerHTML`, `document.write` — stored **Critical**, reflected High
- Template injection (SSTI): user input rendered as Jinja2/Twig/Pebble template — **Critical**

### 2.2 — Broken Access Control (A01)
- Missing authentication on endpoints handling sensitive data
- IDOR: resource fetched by ID without verifying the authenticated user owns it — **Critical**
- Missing server-side role check on privileged operations — **Critical**
- Mass assignment: object hydrated from `request.json` / `request.body` without allowlisting — High → **Critical**

### 2.3 — Cryptographic Failures (A02)
- Hardcoded credentials, API keys, passwords as literal strings — **Critical**
- Passwords hashed with MD5, SHA1, or SHA256 without salt — **Critical**
- Weak algorithms: DES, 3DES, RC4, ECB mode, RSA < 2048-bit — High
- Non-cryptographic RNG (`random`, `Math.random`) for security values — High

### 2.4 — SSRF (A10)
- User-controlled URL passed to a server-side HTTP function — **Critical** (cloud metadata) / High (internal services)

### 2.5 — Insecure Deserialization (A08)
- Untrusted data in `pickle.loads`, Java `ObjectInputStream`, PHP `unserialize`, YAML `load` (not `safe_load`) — **Critical**

### 2.6 — Path Traversal
- User input in file path construction without sanitization — High (read) → **Critical** (write/execute)

### 2.7 — Security Misconfiguration (A05)
- Debug mode in production; `CORS: *` with credentials; default credentials; `verify=False` in HTTPS requests

---

## Step 3: AI/LLM security pass (skip if no AI code in scope)

Refer to `references/owasp-llm-top-10.md` for full code patterns and mitigations. Quick-reference severity triggers:

| LLM category | Trigger | Severity |
|---|---|---|
| **LLM01 Prompt Injection** | User input or external doc concatenated into prompt + agent has write/delete/exec/send tools | **Critical** |
| **LLM01 Prompt Injection** | Injection with read-only or generation-only impact | High |
| **LLM05 Improper Output Handling** | LLM output → innerHTML / SQL / shell / deserialization | **Critical** |
| **LLM06 Excessive Agency** | Agent with unrestricted file, shell, or network access and no confirmation step | **Critical** |
| **LLM06 Excessive Agency** | Agent with overly broad read scope | High |
| **LLM02 Sensitive Info Disclosure** | Credentials or session tokens in system prompt | **Critical** |
| **LLM02 Sensitive Info Disclosure** | PII in prompt logged to accessible location | High |
| **LLM07 System Prompt Leakage** | System prompt contains credentials | **Critical** |
| **LLM07 System Prompt Leakage** | System prompt contains confidential business logic | High |
| **LLM08 Vector/Embedding Weaknesses** | Vector store returns other users' private data | **Critical** |
| **LLM08 Vector/Embedding Weaknesses** | Missing namespace isolation for sensitive documents | High |
| **LLM10 Unbounded Consumption** | LLM endpoint unauthenticated + no `max_tokens` + no rate limit | **Critical** |
| **LLM10 Unbounded Consumption** | Missing rate limits behind authentication | High |

---

## Step 4: Assign severity and confidence

Use `references/severity-rubric.md` for authoritative definitions. Summary:

| Severity | Criteria |
|---|---|
| **Critical** | Exploitable with no special privileges or minimal conditions; full compromise, mass data breach, RCE, or irreversible damage |
| **High** | Exploitable under realistic conditions; significant breach, privilege escalation, or serious disruption |
| **Medium** | Limited impact or difficult to exploit |
| **Low** | Minor; defense-in-depth gap |

Confidence levels:
- **Confirmed** — direct evidence (code line + data flow)
- **Likely** — strong indicator, one hop removed
- **Needs human review** — pattern matches but context ambiguous

Only surface Critical/High with Confirmed or Likely confidence.

---

## Step 5: Write the security review document

Get the branch name:
```bash
git rev-parse --abbrev-ref HEAD
```

Create `docs/security/review-<branch-name>.md`. Use `templates/security-review.md` as the structure. If `docs/security/` doesn't exist, create it.

Key sections:
- Scope: files reviewed / not reviewed
- Critical Findings: title, severity, confidence, category (OWASP ref), file:line, evidence snippet, exploitation scenario, recommended fix
- High Findings: same structure
- Suppressed Findings count
- Coverage Notes: what was NOT reviewed and why
- Next Steps

Header attribution: `**Reviewer:** cyber-champ / security-review skill`

---

## Step 6: Escalate Critical/High findings needing security team input

If any Critical or High finding:
- Has no clear, safe mitigation path
- Requires an architecture decision the team can't make alone
- Involves a compliance question (GDPR, financial regulation, health data)

Invoke `/security-triage` or create `docs/security/escalation-<branch-name>.md` directly.

---

## Behaviour guidelines

- **Cite file:line for every finding.** Do not flag "potential SQL injection" without the specific line.
- **Give concrete fixes.** Not "use parameterized queries" but "replace line 42 with `cursor.execute('SELECT * FROM users WHERE id = %s', (user_id,))`."
- **Acknowledge scope.** State clearly what you didn't review. Do not claim "no security issues" for code you haven't read.
- **Don't guess.** If you see a call to an unknown helper, note it and mark confidence as "Needs human review" rather than assuming safe or vulnerable.
- **Don't over-flag.** If a pattern looks like injection but the input is clearly a compile-time constant, don't flag it.
