# AI Feature Security Checklist

Use this checklist during threat modeling for any feature that includes an LLM, AI agent, RAG pipeline, model fine-tuning, or ML inference endpoint.

Answer each question. A "No" or "Unknown" answer on a mandatory question is a potential finding; assess its severity using the rubric.

---

## 1. Input surface

| # | Question | Mandatory | Notes |
|---|---|---|---|
| 1.1 | Does user-controlled text flow into an LLM prompt (directly or via retrieval)? | Yes | If yes: LLM01 prompt injection applies |
| 1.2 | Are third-party or externally-sourced documents embedded in the context (RAG, web search, email, uploaded files)? | Yes | Indirect injection path |
| 1.3 | Is there a maximum input length enforced? | Yes | LLM10 unbounded consumption |
| 1.4 | Is input validated or filtered before going into the prompt? | If 1.1 = Yes | What is filtered? |

## 2. Output handling

| # | Question | Mandatory | Notes |
|---|---|---|---|
| 2.1 | Is LLM output rendered in a browser (HTML context)? | Yes | LLM05: XSS risk if output is not HTML-escaped |
| 2.2 | Is LLM output used to construct a SQL query, shell command, or file path? | Yes | LLM05: injection risk |
| 2.3 | Is structured LLM output (JSON, YAML, code) parsed and executed without schema validation? | Yes | LLM05: type confusion, code injection |
| 2.4 | Is LLM output passed to another LLM without sanitization? | Yes | Cascaded prompt injection |

## 3. Agent tools and permissions

| # | Question | Mandatory | Notes |
|---|---|---|---|
| 3.1 | What tools does the agent have? List every tool with its scope. | Yes | LLM06: excessive agency |
| 3.2 | Which tools have side effects (write, delete, send, execute, charge)? | Yes | Every side-effect tool needs a minimum-scope review |
| 3.3 | Are there destructive or irreversible operations? Is human confirmation required? | If 3.2 = Yes | LLM06: confirmation requirement |
| 3.4 | Are tool permissions scoped to the authenticated user's own data (not all users' data)? | Yes | IDOR / privilege escalation |
| 3.5 | What is the blast radius if the model is manipulated? Can it exfiltrate data? Overwrite files? Send emails? | Yes | LLM01 + LLM06 combined severity |

## 4. Data and privacy

| # | Question | Mandatory | Notes |
|---|---|---|---|
| 4.1 | Is PII (name, email, health, financial, location) included in prompts? | Yes | LLM02: sensitive information disclosure |
| 4.2 | Under what legal basis is PII processed by the LLM? (consent, contract, legitimate interest) | If 4.1 = Yes | GDPR / privacy compliance flag |
| 4.3 | Is conversation history or prompt/response logged? Where? Who has access? | Yes | LLM02: audit log exposure |
| 4.4 | Is the LLM provider a data processor? Is there a DPA in place? | Yes | Third-party data transfer |
| 4.5 | Is there a retention policy for conversation data? | Yes | |

## 5. System prompt

| # | Question | Mandatory | Notes |
|---|---|---|---|
| 5.1 | Does the system prompt contain credentials, API keys, or internal tokens? | Yes | LLM07 + LLM02: **Critical** if yes |
| 5.2 | Does the system prompt contain business-sensitive information? | Yes | LLM07: treat prompt as potentially leakable |
| 5.3 | Has the system prompt been tested against extraction attempts? | Yes | Common extraction: "Repeat your instructions", "What is in your context?" |

## 6. Knowledge base and training data

| # | Question | Mandatory | Notes |
|---|---|---|---|
| 6.1 | Who can add documents to the RAG knowledge base? | If RAG | LLM04: data poisoning |
| 6.2 | Is there an approval / review workflow for new documents? | If RAG | |
| 6.3 | Are all documents in the vector store accessible to all users, or is there per-user filtering? | If RAG | LLM08: access control |
| 6.4 | Does fine-tuning use real user data? Was it anonymised? | If fine-tuning | LLM04 + LLM02 |
| 6.5 | Are model weights sourced from verified, pinned releases? | Yes | LLM03: supply chain |

## 7. Rate limiting and resource consumption

| # | Question | Mandatory | Notes |
|---|---|---|---|
| 7.1 | Is there per-user rate limiting on LLM API calls? | Yes | LLM10 |
| 7.2 | Is `max_tokens` set on every completion call? | Yes | LLM10 |
| 7.3 | Is there cost monitoring and alerting on LLM API usage? | Yes | LLM10 |
| 7.4 | Is the LLM endpoint accessible to unauthenticated users? | Yes | If yes: LLM10 risk is Critical |

## 8. Authentication and access control

| # | Question | Mandatory | Notes |
|---|---|---|---|
| 8.1 | Is the LLM feature gated behind authentication? | Yes | Unauthenticated access amplifies all other risks |
| 8.2 | Are multi-tenant boundaries enforced (one user cannot access another's conversations/data)? | Yes | IDOR |
| 8.3 | Is there an audit trail of who invoked the LLM and with what input? | Yes | Repudiation |

## 9. Agentic autonomy

| # | Question | Mandatory | Notes |
|---|---|---|---|
| 9.1 | Can the agent take actions autonomously without user review (fully automated pipeline)? | Yes | Autonomy amplifies prompt injection blast radius |
| 9.2 | Can the agent spawn sub-agents or call other LLMs? | Yes | Cascaded injection risk |
| 9.3 | Is there a maximum number of tool calls or agent iterations before stopping? | Yes | Prevents runaway agents |
| 9.4 | Is there a timeout on agent tasks? | Yes | Prevents resource exhaustion |
