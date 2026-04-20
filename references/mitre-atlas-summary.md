# MITRE ATLAS — AI/ML Threat Summary

Source: MITRE ATLAS (Adversarial Threat Landscape for Artificial-Intelligence Systems). Focus on techniques relevant to production AI features.

---

## Most Relevant Techniques for Threat Modeling

### AML.T0054 — LLM Prompt Injection

**What it is:** Adversary crafts input that overrides LLM instructions to redirect behavior, extract information, or trigger unauthorized actions.

**Sub-techniques:**
- Direct injection via user input
- Indirect injection via retrieved context (documents, search results, emails)
- Multi-turn injection across conversation history

**Threat model relevance:** Any feature where user input or external data reaches an LLM prompt. Severity amplified when the LLM has tools with side effects.

**OWASP LLM cross-reference:** LLM01, LLM06.

---

### AML.T0051 — LLM Jailbreak

**What it is:** Adversary uses crafted prompts to bypass the model's safety guardrails, causing it to produce harmful content, reveal training data, or act outside its intended scope.

**Threat model relevance:** Consumer-facing features where the LLM is expected to refuse certain requests. Also relevant when the LLM makes security-relevant decisions (e.g., "should I perform this action?").

**Mitigation focus:** Do not rely on the LLM itself as the only enforcement mechanism for security policies. Enforce in application code, not in the prompt.

---

### AML.T0043 — Craft Adversarial Data (Evasion)

**What it is:** Adversary perturbs input data (images, text) to cause the model to misclassify or produce incorrect outputs while appearing normal to humans.

**Threat model relevance:** Content moderation, spam detection, fraud detection, identity verification features built on ML models. An adversary who can test the model can learn to evade it.

**Severity trigger:** If a model's misclassification has security consequences (e.g., evading fraud detection, bypassing content moderation) → High.

---

### AML.T0048 — Backdoor ML Model

**What it is:** Adversary introduces a backdoor into a model during training — a specific trigger pattern causes the model to behave maliciously while appearing normal otherwise.

**Threat model relevance:** Fine-tuning on untrusted datasets; using models from unverified third-party sources; accepting user-submitted training data.

**OWASP LLM cross-reference:** LLM04 (Data and Model Poisoning).

---

### AML.T0049 — Craft Adversarial Data (Poisoning)

**What it is:** Adversary injects malicious examples into the training pipeline to cause the trained model to learn incorrect behaviors.

**Threat model relevance:** Products that allow user-contributed training data, fine-tuning pipelines, RAG knowledge base ingestion from untrusted sources.

---

### AML.T0040 — ML Model Inference API Access

**What it is:** Adversary uses repeated queries to the model's inference API to extract information about the model's behavior, training data, or to reconstruct the model.

**Threat model relevance:** Public-facing model APIs. Membership inference attacks can reveal whether specific records were in the training set — relevant for privacy.

---

### AML.T0044 — Full ML Model Access

**What it is:** Adversary obtains direct access to model weights, enabling offline attacks (parameter extraction, inversion attacks to recover training data).

**Threat model relevance:** Model artifacts stored in accessible storage (S3 buckets, CDN), model weights checked into public repos, LLM fine-tunes deployed without access control.

**Severity trigger:** Model weights in publicly accessible storage → **Critical** if model was fine-tuned on private data.

---

### AML.T0056 — LLM Meta-Prompt Extraction

**What it is:** Adversary uses direct queries or multi-turn conversation to extract the contents of the system prompt or hidden instructions.

**OWASP LLM cross-reference:** LLM07.

---

## ATLAS × STRIDE Quick Reference

| ATLAS Technique | STRIDE Category | When it applies |
|---|---|---|
| AML.T0054 Prompt Injection | S, I, E | LLM with user input or RAG |
| AML.T0051 Jailbreak | E | Consumer LLM feature |
| AML.T0043 Adversarial Evasion | T | ML classification in security context |
| AML.T0048 Backdoor | T | Fine-tuning, model supply chain |
| AML.T0049 Poisoning | T | RAG ingestion, training pipelines |
| AML.T0040 Model Inference | I | Public model API |
| AML.T0044 Full Model Access | I | Model storage |
| AML.T0056 Prompt Extraction | I | System prompt with sensitive content |
