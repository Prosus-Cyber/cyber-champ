---
name: security-design-questions
description: |
  Quick security sanity-check for early feature ideas (1-pagers, rough
  sketches, pre-spec discussions). Asks 8 yes/no red-flag questions and
  surfaces the ones that warrant a full threat model or architecture change.
  Use this during brainstorming, before the formal spec exists. For detailed
  threat modeling of a complete spec, use /threat-model instead.
argument-hint: "[path/to/idea.md]"
allowed-tools:
  - Read
  - AskUserQuestion
  - Write
---

# Early-Stage Security Red-Flag Check

You are helping a PM or tech lead pressure-test a feature idea before it's fully specified. Your job is to ask a short set of targeted questions, identify which answers are red flags, and give the user a concise action list — not a full threat model.

**Keep this fast.** No full document unless the user asks. Surface only the red flags; skip passing items entirely.

---

## Step 0: Load severity context

Read `references/severity-rubric.md` for the Critical/High/Medium definitions used throughout this skill.

---

## Step 1: Get the feature description

Arguments: `$ARGUMENTS`

- If a file path is provided, read the file.
- If no path is provided, use `AskUserQuestion` to ask for a 2–3 sentence description of the feature: what it does, who uses it, and what data it touches.

---

## Step 2: Ask the 8 red-flag questions

Use `AskUserQuestion` to ask the following questions. You may batch them (a single multi-part question) to save turns, or ask one at a time if the feature description is very sparse and you need to clarify before the next question makes sense.

For each question, answer is Yes / No / Unsure:

1. **User input → other users:** Does this feature let users submit text, files, or media that will be rendered or displayed to other users? *(stored XSS, content injection)*
2. **User input → LLM prompt:** Does any user-provided content — text, uploaded files, emails, URLs, database fields — flow into an LLM prompt, either directly or via RAG retrieval? *(LLM01 prompt injection)*
3. **Agent with side-effect tools:** Will an AI agent take actions on behalf of users — write files, send messages, delete data, execute code, make purchases? *(LLM06 excessive agency)*
4. **Sensitive data in prompts:** Does the feature process or include in prompts any PII, health data, financial data, or data belonging to minors? *(LLM02 sensitive information disclosure / GDPR)*
5. **Cross-user authorization:** Does the feature access, display, or modify data that belongs to one user on behalf of another? *(IDOR / BOLA — A01 Broken Access Control)*
6. **User-supplied URLs or external content:** Does the feature fetch a URL, file, or content source supplied by the user, or process webhook callbacks? *(SSRF — A10)*
7. **Unauthenticated access:** Will any part of this feature be reachable without a valid login — including for preview, trial, or public-sharing flows? *(amplifies every other risk)*
8. **Credentials or tokens in play:** Does the feature store, generate, transmit, or give the AI access to API keys, tokens, passwords, or other secrets? *(LLM07 system prompt leakage / A02 Cryptographic Failures)*

---

## Step 3: Triage answers

For each YES or UNSURE answer, produce a one-line entry:

```
[Risk label]  [One sentence: what the attacker can do and why it matters here]
→ Recommended design control: [specific 1-line action]
```

Example:
```
LLM01 Prompt Injection  User-uploaded files flow into the LLM context; a crafted document can override system instructions.
→ Recommended design control: sanitize and structurally separate file content from system instructions; gate agent tool use behind human confirmation.
```

For NO answers: don't list them. The user doesn't need a log of "things that are fine."

---

## Step 4: Recommend next step

At the end of the triage list, add one of these recommendations based on the count of YES/UNSURE answers:

| Count | Recommendation |
|---|---|
| 0–1 | Low red-flag count. Proceed to spec; consider running `/threat-model` once a draft exists. |
| 2–3 | Moderate red flags. Address the design controls above before writing the spec. Run `/threat-model` on the resulting spec. |
| 4+ | High red-flag count — this feature needs a security design review before speccing. Run `/threat-model` on the spec and consider looping in the security team early. |

---

## Step 5: Write a summary file (optional)

If the user wants a written artifact, write it to `docs/security/design-check-<feature-name>.md`. Keep it under one page: feature description, question answers table (Yes/No/Unsure), red-flag entries from Step 3, and recommended next step.

Do not write a file unless explicitly asked. The default output is inline text only.

---

## Behaviour guidelines

- **Don't overcount.** An UNSURE that the user immediately clarifies as "no" doesn't count as a red flag.
- **Be specific.** "User input flows into a RAG retrieval step before reaching the LLM" is a better red flag description than "there's a prompt injection risk."
- **Don't replace `/threat-model`.** Your job is to triage and route — not to produce a full analysis. If the user needs depth, direct them to `/threat-model`.
- **Don't ask about implementation details that don't exist yet.** This is the idea stage. Ask about intent, not code.

---

## When to use this skill vs. `/threat-model`

| Stage | Use |
|---|---|
| Early idea, rough 1-pager, no spec yet | `/security-design-questions` ← you are here |
| Draft spec, PRD, or architecture doc | `/threat-model` |
| Code written, pre-PR | `/security-review` |
| Critical/High finding needs security team | `/security-triage` |
