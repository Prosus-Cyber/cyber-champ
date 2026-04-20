<p align="center">
  <img src="cyber-champ.png" alt="Cyber Champion logo" width="350">
</p>

# Cyber Champion (`cyber-champ`)

A Claude Code plugin for self-service security assessment. Helps developers and product managers identify Critical and High security issues — including AI-specific threats — without requiring the security team to be involved in every feature.

Covers: traditional threats (OWASP Top 10, API Security, MASVS) and AI-specific threats (OWASP LLM Top 10 2025, MITRE ATLAS).

---

## Skills

| Skill | For | When to use | Output |
|---|---|---|---|
| `/security-design-questions` | PMs / tech leads | Brainstorming a rough idea, before a formal spec exists | Inline red-flag summary |
| `/threat-model` | PMs / tech leads | Drafting a feature spec or PRD | `docs/security/threat-model-<feature>.md` |
| `/security-review` | Developers | Before opening a PR | `docs/security/review-<branch>.md` |
| `/security-triage` | Either | A Critical/High finding needs security team input | `docs/security/escalation-<name>.md` |

All four skills also auto-invoke when Claude detects the relevant context (rough feature ideas, spec files, security-sensitive code).

### When to use which skill

```
Rough idea / 1-pager
  → /security-design-questions    (5-min sanity check, 8 yes/no questions)
        ↓ (3+ red flags or spec written)
Full spec or PRD
  → /threat-model                 (full STRIDE + AI checklist, written report)
        ↓
Code written, pre-PR
  → /security-review              (code-level pattern scan, written report)
        ↓ (Critical/High with no clear fix)
Security team needed
  → /security-triage              (structured escalation report)
```

---

## Installation

### Option A: Install as a plugin (recommended)

```bash
# From the Claude Code CLI
/plugin install /path/to/cyber-champ
```

Or for the whole team, add to your repo's `.claude/plugins/`:
```bash
cp -r cyber-champ /path/to/your/repo/.claude/plugins/
```
Commit the `.claude/plugins/cyber-champ/` directory so the whole team gets it automatically.

### Option B: Install as a skill bundle

Copy into the global skills directory:
```bash
cp -r cyber-champ ~/.claude/skills/
```

Or per-repo:
```bash
cp -r cyber-champ /path/to/your/repo/.claude/skills/
```

### Option C: Symlink (for active development of the plugin)

```bash
ln -s /path/to/cyber-champ ~/.claude/skills/cyber-champ
```

---

## Usage

### Early idea check — for product managers

```
/security-design-questions
```

Claude will ask you 8 yes/no questions about the feature idea and surface any red flags. If 3+ are flagged, it will recommend running `/threat-model` once the spec is written.

### Threat modeling a feature spec

```
/threat-model @docs/specs/my-feature.md
```

Output: `docs/security/threat-model-my-feature.md`

**Show all severities (not just Critical/High):**
```
/threat-model verbosity: all @docs/specs/my-feature.md
```

### Reviewing code before a PR

```
/security-review
```

Reviews all files changed on the current branch vs. the base branch (auto-detects `main`, `master`, `develop`, `trunk`).

**Review specific files:**
```
/security-review @src/api/routes/auth.ts @src/lib/llm.ts
```

**Show all severities:**
```
/security-review verbosity: all
```

Output: `docs/security/review-<branch>.md`

### Escalating to the security team

The `threat-model` and `security-review` skills automatically create an escalation report when they find a Critical/High issue without a clear mitigation. You can also invoke manually:

```
/security-triage @docs/security/review-my-branch.md finding: FINDING-001
```

Output: `docs/security/escalation-<name>.md` — send this to the security team.

---

## PR workflow integration

Add this to your PR template:

```markdown
## Security
- [ ] Ran `/security-review` before opening this PR
- [ ] Link to security review: `docs/security/review-<branch>.md`
  - [ ] No Critical or High findings, OR Critical/High findings documented and addressed/escalated
```

---

## Default behaviour: Critical and High only

By default, all skills surface **Critical and High** findings only. Medium, Low, and Info findings are computed but suppressed. A footer in every report shows the count of suppressed findings.

This is intentional: surfacing every minor issue trains developers to ignore the output. Critical and High are the findings that must be addressed before shipping.

To see all severities: add `verbosity: all` to any invocation.

---

## Knowledge base

The `references/` directory is the authoritative source behind the skill logic. Skills read these files at runtime — updating a reference file immediately changes what Claude looks for and flags. Update these files when standards are revised:

| File | Content |
|---|---|
| `references/severity-rubric.md` | Critical/High/Med/Low/Info definitions and suppression policy |
| `references/owasp-llm-top-10.md` | OWASP LLM Top 10 2025 with code patterns |
| `references/ai-feature-checklist.md` | Design-time AI security checklist |
| `references/stride-worksheet.md` | STRIDE framework with AI-specific questions |
| `references/owasp-top-10-web.md` | OWASP Top 10 Web 2021 |
| `references/owasp-api-top-10.md` | OWASP API Security Top 10 2023 |
| `references/owasp-masvs-mobile.md` | OWASP MASVS mobile security |
| `references/mitre-atlas-summary.md` | MITRE ATLAS AI/ML threat techniques |
| `references/nist-ai-rmf-summary.md` | NIST AI RMF governance questions |
| `references/escalation-template.md` | Escalation report format reference |

---

## Calibration (recommended before rollout)

Before deploying org-wide:

1. **Threat-model calibration:** Run `/threat-model` on 3 recent feature specs. Compare Critical/High output against what the security team would have flagged. Tune `references/severity-rubric.md` if the boundary is wrong.
2. **Code-review calibration:** Run `/security-review` on 5 recent PRs the security team reviewed. Verify all their Critical/High findings are captured. Check false-positive rate (goal: fewer than 30% false positives on Critical/High findings).
3. **AI red-team test:** Write a deliberately vulnerable AI feature (prompt injection sink, excessive agent scope, PII in prompt). Confirm both skills flag the AI-specific issues by OWASP LLM category.

Re-calibrate quarterly as the threat landscape evolves. Update reference files only — the skills don't need to change.

---

## Scope

**In scope:** Threat modeling, code review (traditional + AI-specific), escalation reporting, early-stage security design questions.

**Out of scope (use dedicated tools):**
- Secrets scanning → GitGuardian, truffleHog, detect-secrets
- Dependency CVE scanning → Dependabot, Snyk, `npm audit`, `pip audit`
- DAST / runtime scanning → OWASP ZAP, Burp Suite
- Privacy/compliance automated checks → dedicated DLP or compliance tooling

---

## References

- [OWASP LLM Top 10 2025](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [OWASP Top 10 Web 2021](https://owasp.org/www-project-top-ten/)
- [OWASP API Security Top 10 2023](https://owasp.org/www-project-api-security/)
- [OWASP MASVS](https://mas.owasp.org/MASVS/)
- [MITRE ATLAS](https://atlas.mitre.org/)
- [NIST AI RMF](https://www.nist.gov/system/files/documents/2023/01/26/AI%20RMF%201.0.pdf)
- [Google SAIF](https://saif.google/)
