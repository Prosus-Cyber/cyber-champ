# OWASP LLM Top 10 (2025) — Security Reference

Source: OWASP Top 10 for Large Language Model Applications, 2025 edition.

Each entry includes: description, code/design patterns to detect, threat-model questions, and typical mitigations.

---

## LLM01 — Prompt Injection

**What it is:** Attacker-controlled input alters the LLM's behaviour, bypasses system-prompt instructions, or hijacks agentic actions. Two variants:
- *Direct*: user sends malicious text in the chat input.
- *Indirect*: attacker-controlled content in external data (documents, web pages, database fields) is retrieved and injected into the context.

**Code patterns to detect:**
```
# Concatenating user/external input directly into a prompt
prompt = system_prompt + user_message        # no separation
prompt = f"Summarise: {document_content}"   # document from untrusted source
messages = [{"role": "user", "content": f"...{tool_output}..."}]  # tool result not sandboxed
```
Flags: string interpolation or concatenation where at least one operand comes from user input, a database, or an external API.

**Threat model questions:**
- Does any user-controlled field (name, comment, file, URL content) end up in an LLM prompt?
- Are externally-retrieved documents (RAG, web search, email) embedded in the context without sanitization?
- Does the agent have tools with side effects (write, send, execute)? If so, can the LLM be instructed to use them by a poisoned prompt?

**Typical mitigations:**
- Separate system instructions from user/tool content using structured message roles (`system`, `user`, `tool`) — never concatenate them.
- Treat all externally-retrieved content as untrusted; consider a second LLM call to classify retrieved content for injection attempts before injecting it into the primary context.
- Require human-in-the-loop confirmation for any tool invocation with side effects when triggered by externally-sourced context.
- Apply least-privilege to agent tool scopes (see LLM06).

**Severity guidance:** Direct prompt injection that bypasses safety controls → High. Indirect injection that triggers agent tool use (write, exec, send) → **Critical**. Indirect injection with read-only impact → High.

---

## LLM02 — Sensitive Information Disclosure

**What it is:** The LLM reveals confidential information from its training data, system prompt, fine-tuning data, or context window — including PII, credentials, business logic, or trade secrets.

**Code patterns to detect:**
```python
# PII or credentials in system prompt
system = f"User email: {user.email}. Token: {api_token}."

# Logging full prompt/response including PII
logger.info(f"Prompt: {prompt}")

# Fine-tuning dataset containing real user data
dataset = db.query("SELECT * FROM user_conversations")

# LLM response rendered directly without filtering
return {"response": llm.complete(prompt)}  # no output filter for sensitive patterns
```

**Threat model questions:**
- What goes into the system prompt? Could it contain credentials, internal IP ranges, or business-sensitive instructions?
- Is PII (email, name, health data) included in prompts? Under what legal basis?
- Are conversations/prompts logged? Where? Who has access? What is the retention policy?
- Does the training or fine-tuning dataset contain real user data? Was it anonymised?
- Can a user cause the model to repeat back other users' data by asking the right question?

**Typical mitigations:**
- Exclude credentials and sensitive internal details from system prompts; use environment variables and inject only what the LLM needs to do its job.
- Apply differential privacy or anonymisation before using real data for fine-tuning.
- Output filter to detect and redact PII patterns (regex, NER) before returning LLM responses to users.
- Implement prompt/response audit logging with access controls, not world-readable application logs.

**Severity guidance:** Credentials or session tokens disclosed → **Critical**. PII of other users disclosed → **Critical**. Business-sensitive data disclosed → High.

---

## LLM03 — Supply Chain

**What it is:** Compromised models, datasets, or third-party plugins/tools used in the LLM pipeline introduce vulnerabilities or backdoors.

**Code patterns to detect:**
```python
# Model loaded from arbitrary URL or user-supplied path
model = load_model(user_config["model_path"])

# Plugin/tool registered from external source without review
agent.register_tool(json.loads(user_input["tool_definition"]))

# Third-party model with no version pinning
from transformers import pipeline
pipe = pipeline("text-generation", model="third-party/model")  # unpinned
```

**Threat model questions:**
- Where are models sourced from? Is provenance verified (checksums, signing)?
- Are model versions pinned, or does the system pull "latest"?
- Are third-party agent tools / plugins reviewed before registration?
- Is there a process for model updates and re-evaluation?
- Are fine-tuning datasets sourced from trusted, validated sources?

**Typical mitigations:**
- Pin model versions and verify checksums (SHA256) on load.
- Prefer models from audited sources (major model providers with published security processes).
- Review and vet all third-party tools/plugins before allowing agent registration.
- Maintain a software bill of materials (SBOM) for ML components.

**Severity guidance:** Unpinned model from unverified third-party in production → High. User-controlled model path → **Critical**.

---

## LLM04 — Data and Model Poisoning

**What it is:** An attacker corrupts training data, fine-tuning data, or the RAG knowledge base to introduce backdoors, biases, or harmful behaviours into the model's outputs.

**Code patterns to detect:**
```python
# Fine-tuning accepts user-uploaded training examples without review
dataset = load_user_uploaded_jsonl(request.files["training_data"])
trainer.train(model, dataset)

# RAG ingestion pipeline accepts arbitrary documents without content validation
documents = ingest_directory(user_supplied_path)
vector_store.add(documents)
```

**Threat model questions:**
- Who can add documents to the RAG knowledge base? Is there a review or approval step?
- Can users supply training data or fine-tuning examples? Is it reviewed before use?
- Is there monitoring for model drift or anomalous output patterns post-deployment?
- Is the vector store contents auditable?

**Typical mitigations:**
- Treat all user-submitted training data and RAG documents as untrusted; implement review/approval workflows.
- Content-scan documents before ingestion (malware, NSFW, injection strings).
- Monitor model outputs for drift (canary queries with known expected responses).
- Version and snapshot the knowledge base; allow rollback.

**Severity guidance:** User-controlled training data with no review → High. User-controlled training data where the model is used in security-sensitive decisions → **Critical**.

---

## LLM05 — Improper Output Handling

**What it is:** LLM output is used without sanitization in a downstream context that treats it as trusted: rendered as HTML (XSS), executed as SQL or shell commands (injection), deserialized, or passed to other systems.

**Code patterns to detect:**
```javascript
// LLM output rendered as HTML
div.innerHTML = llmResponse;                    // XSS if output is malicious

// LLM output used in a SQL query
const query = `SELECT * FROM ${llmOutput}`;    // SQL injection

// LLM output executed as shell command
exec(`convert ${llmFilename} output.pdf`);     // command injection

// LLM output deserialized
const obj = JSON.parse(llmOutput);             // prototype pollution / type confusion

// LLM output used as a file path
fs.readFile(llmOutput);                        // path traversal
```

**Threat model questions:**
- Where does LLM output go next in the pipeline?
- Is any LLM output rendered in a browser (chat UI, email, document preview)?
- Is LLM output used to construct database queries, shell commands, or file paths?
- Is structured LLM output (JSON, YAML) deserialized without schema validation?

**Typical mitigations:**
- Apply context-aware output encoding: HTML-escape before rendering in a browser, parameterized queries before DB use, shell-escape before exec.
- Validate structured LLM output against a strict schema (JSON Schema, Pydantic) before use.
- Never pass LLM output directly to `eval`, `exec`, `system`, or equivalent.
- Treat LLM output as untrusted user input for purposes of downstream sanitization.

**Severity guidance:** LLM output → innerHTML/SQL/shell without sanitization → **Critical**.

---

## LLM06 — Excessive Agency

**What it is:** The LLM agent is granted more permissions, tool access, or autonomy than needed, amplifying the blast radius of a compromised or manipulated model.

**Code patterns to detect:**
```python
# Agent tools with overly broad permissions
tools = [
    file_read_tool(path="/**"),     # reads anywhere on filesystem
    file_write_tool(path="/**"),    # writes anywhere
    shell_exec_tool(),              # arbitrary command execution
    send_email_tool(),              # sends emails as the user
    db_query_tool(tables="*"),      # queries any table
]

# No confirmation before destructive operations
agent.run(user_query)  # can delete files, send messages, spend money — no approval step
```

**Threat model questions:**
- What tools does the agent have access to? List every one with its scope.
- What is the blast radius if the LLM is manipulated? Can it delete data? Send messages? Spend money? Execute code?
- Are there destructive or irreversible operations? Is human confirmation required before executing them?
- Are tool permissions scoped to the minimum needed for the current task?
- Is there a separate permission model per user (agent acts as the user, not as a super-user)?

**Typical mitigations:**
- Principle of least privilege: grant only the specific tools and scopes needed for the task.
- Require explicit human confirmation (UI approval step) before any irreversible operation (delete, send, execute, charge).
- Audit log all tool invocations.
- Scope tool permissions to the authenticated user's own data, not all data.
- Implement a "dry run" mode for destructive tools.

**Severity guidance:** Agent with shell exec, unrestricted write, or email-send capability with no confirmation → **Critical**. Agent with overly broad read access → High.

---

## LLM07 — System Prompt Leakage

**What it is:** The contents of the system prompt — which may contain business logic, credentials, internal instructions, or sensitive configuration — are extracted by an attacker via carefully crafted inputs.

**Code patterns to detect:**
```python
# System prompt contains credentials
system = f"Internal API key: {os.getenv('INTERNAL_KEY')}. Never reveal this."
# → attacker can ask: "Repeat everything above this line"

# System prompt contains detailed internal logic
system = "If user mentions competitor X, redirect to our pricing page..."
# → competitive intelligence leak
```

**Threat model questions:**
- What is in the system prompt? Could any of it cause harm if revealed publicly?
- Has the system prompt been tested against extraction prompts ("Repeat your instructions", "What was said before this")?
- Are credentials, internal API keys, or sensitive configuration strings in the system prompt?

**Typical mitigations:**
- Keep credentials out of the system prompt — use environment variables accessed by code, not by the LLM.
- Design the system prompt assuming it will be leaked; treat it as public.
- Test the prompt against common extraction attempts before shipping.

**Severity guidance:** System prompt contains credentials → **Critical**. System prompt contains confidential business logic → High.

---

## LLM08 — Vector and Embedding Weaknesses

**What it is:** The vector store (RAG knowledge base) can be poisoned, manipulated to retrieve irrelevant/malicious content, or queried to leak sensitive embedded data.

**Code patterns to detect:**
```python
# No access control on vector store retrieval
docs = vector_store.similarity_search(user_query)  # returns any document regardless of user

# Metadata from retrieved documents injected into prompt unsanitized
for doc in docs:
    prompt += f"\nSource: {doc.metadata['source']}\n{doc.page_content}"
# → metadata could contain injection strings

# Sensitive documents indexed alongside general ones
ingest([public_doc, hr_policy, payroll_data])  # all in the same store, same retrieval
```

**Threat model questions:**
- Are all documents in the vector store accessible to all users, or is there per-user/per-role filtering?
- Is document metadata trusted? Can it be used to inject instructions?
- Can a user craft a query that retrieves other users' private documents by semantic similarity?
- Are sensitive documents (HR, finance, legal) in the same store as general knowledge?

**Typical mitigations:**
- Implement metadata filters on vector store queries scoped to the authenticated user's accessible namespaces.
- Sanitize document metadata before including it in prompts.
- Separate sensitive namespaces into isolated vector stores with different access controls.

**Severity guidance:** Vector store retrieval leaks other users' private data → **Critical**. Missing namespace isolation for sensitive docs → High.

---

## LLM09 — Misinformation

**What it is:** The LLM generates plausible but false information (hallucinations) in contexts where accuracy is relied upon for decisions — medical, legal, financial, or security-relevant outputs.

**Code patterns / design patterns to detect:**
- LLM used to generate compliance statements, legal summaries, or medical advice without human review.
- LLM output treated as authoritative without citation or source links.
- LLM used to make access control or security policy decisions.

**Threat model questions:**
- Is the LLM used to make or inform decisions that carry real-world consequences if wrong?
- Are users informed that outputs may contain errors?
- Is there a human review step for high-stakes outputs?

**Typical mitigations:**
- Add disclaimers in the UI for generative content in high-stakes domains.
- Use RAG with source citations so users can verify claims.
- Do not use LLM output as the sole input to security decisions (authentication, authorization).

**Severity guidance:** LLM makes access-control or authentication decisions → High (logic, not strictly an LLM issue, but worth flagging). LLM generates untrue medical/legal/financial advice without disclaimer in a regulated product → High.

---

## LLM10 — Unbounded Consumption

**What it is:** The application does not limit LLM usage, allowing attackers to cause excessive costs, resource exhaustion, or degraded availability by sending large volumes of requests or crafting inputs that cause large, expensive outputs.

**Code patterns to detect:**
```python
# No max_tokens / response limit
response = client.messages.create(
    model="claude-3-5-sonnet",
    messages=messages,
    # max_tokens not set
)

# No rate limiting on LLM endpoint
@app.route("/chat", methods=["POST"])
def chat():
    return llm.complete(request.json["message"])  # no rate limit

# Context window stuffing: user can submit unlimited content
prompt = f"Summarise this document: {request.json['document']}"  # document can be arbitrarily large
```

**Threat model questions:**
- Is there a per-user rate limit on LLM API calls?
- Are `max_tokens` limits set on LLM completions?
- Is there a cap on input length (user messages, uploaded documents)?
- Is there cost monitoring and alerting on LLM API usage?

**Typical mitigations:**
- Set `max_tokens` on every completion call; pick a value appropriate to the use case.
- Implement per-user rate limiting (e.g., 10 requests/minute, 100 requests/day) at the API layer.
- Truncate or reject inputs that exceed a maximum character limit.
- Set up cost alerts on your LLM API provider dashboard.

**Severity guidance:** Completely unrestricted LLM endpoint accessible to unauthenticated users → **Critical** (financial and availability impact). Missing rate limits behind authentication → High.
