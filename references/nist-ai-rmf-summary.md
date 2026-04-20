# NIST AI Risk Management Framework — Summary for Threat Modeling

Source: NIST AI RMF 1.0 (2023). Use this for context and framing during threat models; not a checklist but a set of questions that help identify gaps in AI risk governance.

The NIST AI RMF has four core functions: **GOVERN**, **MAP**, **MEASURE**, **MANAGE**.

---

## Govern — Is there accountability for AI risks?

Questions to ask during threat modeling:
- Is there a designated owner responsible for the security and safety of this AI feature?
- Is there a documented policy on acceptable AI use for this product category?
- Are there defined escalation paths when the AI feature behaves unexpectedly or is abused?
- Is there a documented process for reviewing and updating AI system behavior over time?

**Security team integration:** If no governance structure exists, Critical/High findings from `threat-model` and `security-review` may have no owner to resolve them. Flag this.

---

## Map — Do we understand the risk landscape?

Questions to ask:
- Has the AI feature been characterized for its potential negative impacts? (on users, third parties, the business)
- Are AI-specific threats (prompt injection, poisoning, excessive agency) in scope for your security review process?
- Have you identified all stakeholders affected if the AI feature misbehaves?
- Is there a clear boundary between what the AI is allowed to do and what requires human oversight?

**Connection to threat model:** The MAP function is essentially what a threat model does for an AI feature. If no prior threat model exists, this is the motivation for running one.

---

## Measure — Can we detect when the AI is misbehaving?

Questions to ask:
- Is there monitoring for anomalous model outputs (hallucination spikes, output pattern shifts)?
- Are there canary queries with known expected outputs to detect model drift?
- Is there logging sufficient to detect a prompt injection attack post-hoc?
- Are there metrics for AI-specific abuse (unusual request patterns, cost spikes)?

**Connection to OWASP LLM10:** Measurement is the foundation for detecting unbounded consumption and ongoing attacks.

---

## Manage — Can we respond and recover?

Questions to ask:
- If the AI feature produces a harmful output, can it be rolled back or disabled quickly?
- Is there an incident response playbook specifically for AI-related incidents (model misbehavior, data poisoning, prompt injection attack)?
- Can a specific user's conversation/data be deleted on request (GDPR right to erasure)?
- Is there a rollback path for model updates?

**Connection to escalation:** The security-triage escalation report should include the "Requested action" section aligned with these Manage capabilities.

---

## Key NIST AI RMF Risk Categories (for feature threat models)

| Risk category | Threat model questions |
|---|---|
| **Reliability and robustness** | Can the model be manipulated to fail or produce wrong outputs consistently? |
| **Privacy** | Does the feature comply with privacy regulations? Is PII in training data? |
| **Fairness and bias** | If the model makes decisions affecting people, are there discriminatory failure modes? |
| **Security and resilience** | Can the feature be exploited to harm users or the business? (Maps to OWASP LLM Top 10) |
| **Transparency and explainability** | Can users understand why the AI made a decision? (Regulatory requirement in some contexts) |
| **Accountability** | Who is responsible when the AI causes harm? Is this documented? |
