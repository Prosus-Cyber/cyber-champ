# Severity Rubric

**Default behaviour: report Critical and High only.** Medium, Low, and Info findings are computed but suppressed.

## Severity Levels

| Severity | CVSS (approx) | Decision rule | Examples |
|---|---|---|---|
| **Critical** | 9.0–10.0 | Exploitable with no special privileges or user interaction; leads to full system/data compromise, mass account takeover, or irreversible damage | Unauthenticated RCE; prompt injection that exfiltrates all user data via an LLM agent with write/exec tool access; hardcoded admin credentials checked into a public repo |
| **High** | 7.0–8.9 | Exploitable under realistic conditions (authenticated user, specific input); leads to significant data breach, privilege escalation, or serious service disruption | SQL injection behind login; LLM agent with excessive tool scope (e.g., delete permission when only read is needed); insecure direct object reference leaking other users' records; PII sent to an external LLM without consent or contractual basis |
| **Medium** | 4.0–6.9 | Limited impact or requires difficult-to-meet conditions; moderate data exposure or DoS potential | Missing rate-limiting on a public API endpoint; verbose stack traces in error responses; JWT with 30-day expiry |
| **Low** | 0.1–3.9 | Minor impact; defense-in-depth gaps; negligible direct exploitation risk | Missing security headers (CSP, HSTS); cookie without SameSite attribute; dependency pinned to a minor version with a low-risk CVE |
| **Info** | n/a | No direct security impact; process or best-practice observation | "Consider adding audit logging here"; "Document key rotation schedule" |

## Why Critical and High only by default

Medium, Low, and Info are real observations but:
- They rarely block a feature shipment; they belong in a backlog.
- Surfacing them at the same level as Critical/High trains developers to ignore the tool.
- They are largely covered by automated SAST, DAST, and dependency scanners in CI.

Critical and High are the set the security team agrees must be addressed before shipping.

## Confidence levels

Each finding also carries a confidence indicator:

| Confidence | Meaning |
|---|---|
| **Confirmed** | Evidence is direct (code line, data flow trace, or spec statement). The vulnerability is present as described. |
| **Likely** | Strong indicator but one hop removed (e.g., called function appears to lack validation, but not verified in its body). |
| **Needs human review** | Pattern matches but context is ambiguous. Flag for the developer to investigate; do not assert it is definitely a bug. |

Only surface a Critical/High if confidence is Confirmed or Likely. "Needs human review" findings are noted but clearly marked as uncertain.

