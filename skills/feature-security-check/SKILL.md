---
name: feature-security-check
description: |
  Fast Critical/High security check for AI-native consumer features
  before you commit to the approach. Built for PMs and tech leads on
  small, fast-moving teams. Surfaces 0–5 design-time risks with
  consequence-first scenarios and a concrete design choice for each,
  in under 5 minutes. No jargon.
argument-hint: "[@path/to/idea.md | paragraph of feature description] [--save]"
allowed-tools:
  - Read
  - AskUserQuestion
  - Write
---

# Feature Security Check

You are helping a PM or tech lead on a small, fast-moving team pressure-test an AI-native consumer feature before they commit to the approach. Surface only **design-time risks** — things you can't fix later by writing better code: data sensitivity, AI blast radius, third-party vendor retention, default visibility, unauthenticated entry, prompt-injection vectors against private user content.

**Audience assumptions** — apply unless the input clearly contradicts:

- The product is a consumer / end-user app (not B2B SaaS hosting other companies' data).
- It is AI-native: an LLM is in the critical path.
- It does **not** touch health data or government data. If the input mentions those, flag it as out-of-scope and recommend a security-team conversation, but do not run the questionnaire.

**Critical and High only. Soft target 4 questions, hard cap 6. Under 5 minutes wall-clock. No jargon.**

This skill does NOT cover code-level issues (SQL injection, XSS, crypto primitive misuse, dependency CVEs). Those need a code-review tool.

---

## Step 0 — Load severity definitions

Read `references/severity-rubric.md` for the generic Critical/High definitions. The combination lookup that maps answers to severity is inline in Step 4 below.

---

## Step 1 — Get the input

Arguments: `$ARGUMENTS`

Detect a `--save` token anywhere in the arguments and strip it before parsing the rest. Then route:

- **File mode** — argument starts with `@` or resolves to a real file path → Read it.
- **Paste mode** — argument is prose (more than ~40 chars, doesn't resolve to a file) → use the argument directly as the feature description. Most common case (Slack paste, Linear ticket body, Notion paragraph).
- **Interactive mode** — argument empty → use `AskUserQuestion` once for a 2–3 sentence pitch (what it does, who uses it, what data it touches).

If the input clearly involves health data (medical records, patient info, PHI) or government data (passports, SSNs, government IDs), do not run Step 3. Output a single line: *"This skill is scoped to AI-native consumer features. For health/government data, talk to your security team — the design tradeoffs are regulatory, not just technical."* Stop.

---

## Step 2 — Extract a silent sketch

Without surfacing this to the user, extract these five fields from the input. They drive Step 3's adaptive branching and prevent re-asking what's already answered.

1. **Audience** — who is the user (consumers / employees / public)?
2. **Data** — what data classes are involved?
3. **Visibility** — who sees what one user creates?
4. **AI shape** — does the AI just answer text, or does it act on someone's behalf?
5. **External surface** — any unauth path, third-party model/vendor, or fetch of user-supplied URLs?

If a field is unclear from the input, leave it `unknown` and let Step 3 ask for it.

---

## Step 3 — Adaptive question flow

Use `AskUserQuestion` to ask only the questions whose triggers fire. **Skip any question whose answer is already in the sketch.** Drill-downs (Q1b, Q3b) only fire when the parent's answer is too coarse to score accurately. Where the tool supports it, batch a parent question and its drill-down in the same call.

| # | Always / Trigger | Question |
|---|---|---|
| **Q1** (multi-select) | Always | "Which of these does this feature touch? Pick all that apply. (a) **payment data** — card numbers, bank accounts, payouts, (b) **auth secrets** — passwords, session tokens, API keys, (c) **private user content** — DMs, photos, voice recordings, location, journal entries, (d) **biometric** — face, voice, or fingerprint used as identifier, (e) **kids likely** — anyone under 13 plausibly uses this, (f) **only names / emails / display info**, (g) **nothing personal**." |
| **Q1b** | If Q1 includes payment data | "Which payment data specifically? (a) **card numbers / CVV** (PCI scope), (b) **bank or transaction metadata only** — amounts, merchants, timestamps, no card or account numbers." |
| **Q2** | Always | "Who can see what one user creates by default — only them, only their team, anyone with a link, or the whole internet?" |
| **Q3** | If AI in sketch | "Does the AI just *answer* (text out only), or can it *do things* on someone's behalf — send a message, make a payment, delete a record, post publicly, call other systems?" |
| **Q3b** | If Q3 = acts | "What's the worst single action it could take if it got it wrong? (a) **reversible** — a message that can be retracted, a draft a human approves before it ships, (b) **costly but reversible** — spend money or make a payment that can be refunded, (c) **irreversible** — delete records, transfer funds, post publicly, send to external parties." |
| **Q4** | If AI in sketch | "Will the model read content from outside our team's control — user uploads, content the user pastes, web pages it fetches, third-party emails it ingests? Treat anything in those as if a stranger wrote it and could try to redirect the model." |
| **Q5** | Always (if AI + third-party model) | "Does the model vendor (OpenAI, Anthropic, etc.) **retain, log, or train on** the data you send them? Pick: (a) **no — zero-retention / no-train tier confirmed**, (b) **yes — default tier or unclear**, (c) **don't know yet**." |
| **Q6** | If Q2 = link-share or public, OR if input mentions public/share/embed | "Is any part of this reachable without logging in — including a 'preview', 'share link', 'embed', or 'trial' path?" |
| **Q7** | If user-generated content in sketch | "Will users see content created by other users without us reviewing it first?" |

Treat an "Unsure" answer as "yes" for scoring unless the user immediately resolves it to "no."

---

## Step 4 — Score using the combination lookup

Each row fires independently and produces one entry. Rank by severity (Critical first), then by trigger order. Collapse rules with the same root cause and keep the highest severity. Cap output at 5 entries.

**Define HSD (high-sensitivity data) = any of**: card-numbers (Q1b), auth-secrets, private user content, biometric, kids.

**Define external-surface = any of**: Q4 = yes (model reads outside content), Q5 = yes/unsure (vendor retains), Q6 = yes (unauth path reachable).

| Trigger combination | Severity | Title shape |
|---|---|---|
| HSD ∩ Q1 ≠ ∅ AND Q5 = yes (vendor retains/trains) | **Critical** | High-sensitivity data trained or logged by a third-party model vendor |
| HSD ∩ Q1 ≠ ∅ AND Q5 = unsure | **Critical** | High-sensitivity data sent to a vendor whose retention/training policy is unclear |
| HSD ∩ Q1 ≠ ∅ AND Q6 = yes | **Critical** | High-sensitivity data on an unauthenticated path |
| kids ∈ Q1 AND any-external-surface | **Critical** | Under-13 users' data leaving our trust boundary (COPPA exposure) |
| auth-secrets ∈ Q1 AND any-external-surface | **Critical** | Secrets reachable beyond our trust boundary |
| Q3b = irreversible AND Q4 = yes | **Critical** | Agent takes irreversible actions on attacker-controlled input |
| Q3b = costly-reversible AND Q4 = yes | **Critical** | Agent spends money on attacker-controlled input |
| private-content ∈ Q1 AND Q4 = yes | **Critical** | User's private content in the same model context as outside content (exfil via prompt injection) |
| Q2 = public-by-default AND HSD ∩ Q1 ≠ ∅ | **Critical** | Default-public on high-sensitivity data |
| Q3b = reversible AND Q4 = yes | **High** | Agent acts on attacker-controlled input (reversible blast radius) |
| Q1b = transaction-metadata AND (Q4 = yes OR Q5 = yes/unsure) | **High** | Financial metadata leaving our trust boundary (lower than card numbers, still regulated) |
| biometric ∈ Q1 AND any-external-surface | **High** | Biometric identifier leaving our trust boundary |
| private-content ∈ Q1 AND Q5 = yes (and not already Critical for same root cause) | **High** | User's private content trained or logged by a third-party model vendor |
| Q3b = irreversible AND private-content ∈ Q1 (and Q4 = no) | **High** | Agent with irreversible scope on private user content |
| Q6 = yes AND Q7 = yes | **High** | Unauthenticated path serving user-generated content |
| Q7 = yes AND Q2 ∈ {team-only, link-share} | **High** | Users see other users' content without review |
| Q2 = link-share AND Q1 ⊄ {names-only, nothing} | **High** | Link-based sharing default on personal data |
| Q2 = public AND Q1 = {names-only} | **High** | Default-public on user identity |

If zero rows fire, output a single line:

> No Critical/High design risks surfaced. Standard pre-launch security review still applies.

…and stop.

---

## Step 5 — Output

For each surfaced risk, format exactly as:

```
[N] SEVERITY — Title (one phrase, scenario form)
    What happens: One or two sentences describing the concrete attack
    or consequence in THIS feature, not a generic vulnerability
    description.
    Design choice now: One or two sentences describing a decision the
    PM can make today — scope reduction, vendor swap, retention tier,
    human-in-the-loop, default visibility. Not a control to add later.
```

Worked example:

```
[1] CRITICAL — Voice journal entries trained by the model vendor
    What happens: We send users' raw voice journal transcripts to the
    OpenAI default tier, which retains data for 30 days and may use it
    for abuse review. A subpoena or vendor breach exposes private
    diary content users believed was end-to-end ours.
    Design choice now: Switch to OpenAI's zero-retention API tier
    before launch (or self-host an open model for the journaling path
    only). The choice changes the vendor contract — make it before
    you build, not after.
```

Don't list "no" answers. Silence on the safe parts is fine.

If any entry is Critical, append a single trailing line:

> ⚠ If any item is Critical, share this output with whoever owns security on your team before you commit to the approach.

---

## Step 6 — Optional save

If `--save` was in the arguments:

1. Derive `<slug>` from the feature description: kebab-case, ≤30 chars, alphanumeric + hyphens, no leading/trailing hyphen.
2. Write the same Step-5 content to `docs/security/feature-check-<slug>.md`.
3. Confirm the path inline.

Without `--save`, do not write a file.

---

## Behaviour rules

- Soft target 4 questions, hard cap 6. Drill-downs (Q1b, Q3b) only fire when the parent answer is too coarse — never both for sport. If Q1 = {nothing} and there is no AI in the sketch, you should be able to finish in 2 questions.
- Speak in scenarios, never in jargon. Do not say SSRF, IDOR, STRIDE, BOLA, A01, OWASP, CSRF, LLM01, COPPA, PCI, HIPAA, GDPR, or any taxonomy/regulation code in the user-facing output (the lookup names them for our internal logic only — translate to plain language in Step 5).
- Every entry has a *Design choice now* — a decision the PM can make today, not a control for engineering to add later.
- Don't surface Medium or Low items. Don't list "no" answers.
- Don't claim coverage of code-level issues (SQL injection, XSS, crypto primitive misuse, dependency CVEs). If the user asks about those, say this skill is design-stage only and recommend a code-review tool.
- Out-of-scope guard: health and government data. If those appear, stop and recommend the security team (see Step 1).
- This is the idea stage. Ask about intent and data, not implementation details that don't exist yet.
