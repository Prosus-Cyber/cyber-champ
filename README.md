<p align="center">
  <img src="cyber-champ.png" alt="Cyber Champion logo" width="350">
</p>

# Cyber Champion (`cyber-champ`)

A Claude Code plugin for PMs and tech leads on small, fast-moving teams. Surfaces 0–5 Critical/High security risks on an **AI-native consumer feature** before you commit to the approach. No jargon.

The skill flags only **design-time risks** — things you can't fix later by writing better code: data sensitivity, AI blast radius, third-party model retention, default visibility, prompt-injection vectors against private user content.

## Who this is for

- Small teams shipping AI-native consumer / end-user apps.
- **Not** for B2B SaaS hosting other companies' data.
- **Not** for health or government data — talk to a security team directly for those.

## What it does

- Takes a paragraph (Slack, Linear, Notion) or a `@file.md`.
- Asks 2–6 PM-friendly questions.
- Outputs 0–5 ranked Critical/High items, each with *what happens* and a *design choice now* you can decide today.
- Add `--save` to write the report to `docs/security/feature-check-<slug>.md`.

## Install

**Requirements:** [Claude Code](https://claude.com/claude-code) and Git.

Clone the repo:

```bash
cd ~ && git clone https://github.com/Prosus-Cyber/cyber-champ.git
```

In any Claude Code session, register and install:

```
/plugin marketplace add ~/cyber-champ
/plugin install cyber-champ@cyber-champ
```

Verify:

```
/cyber-champ:feature-security-check "Voice journaling app. Users record voice notes; GPT-4 summarizes them into a public mood card."
```

> If `/plugin marketplace add ~/cyber-champ` says path not found, your Claude Code didn't expand `~`. Run `echo ~/cyber-champ` in your terminal to print the full path (e.g. `/Users/yourname/cyber-champ`) and use that instead.

To update later: `cd ~/cyber-champ && git pull` then `/plugin marketplace update cyber-champ`.

## Usage

Paste mode (the usual way):

```
/cyber-champ:feature-security-check "Voice journaling app. Users record voice notes, we transcribe them, GPT-4 turns them into a daily mood summary the user can share to a public link. Using OpenAI's default API tier."
```

From a file:

```
/cyber-champ:feature-security-check @1-pager.md
```

Interactive (the skill asks for a 2–3 sentence pitch first):

```
/cyber-champ:feature-security-check
```

Add `--save` to any of the above to also write `docs/security/feature-check-<slug>.md`.

## Example output

```
[1] CRITICAL — Voice journal entries trained by the model vendor
    What happens: We send users' raw voice journal transcripts to the
    OpenAI default tier, which retains data for 30 days and may use
    it for abuse review. A subpoena or vendor breach exposes private
    diary content users believed was end-to-end ours.
    Design choice now: Switch to OpenAI's zero-retention API tier
    before launch (or self-host an open model for the journaling
    path only). The choice changes the vendor contract — make it
    before you build, not after.

[2] CRITICAL — Default-public mood summary on private content
    What happens: A "share to public link" default on a feature
    backed by private journal entries will quietly publish content
    users would never opt-in to share if asked.
    Design choice now: Make the link share private-by-default, with
    public sharing as an explicit opt-in per summary. Decide now —
    flipping the default later breaks user trust.

⚠ If any item is Critical, share this output with whoever owns
security on your team before you commit to the approach.
```

## Tuning

Edit the combination lookup in `skills/feature-security-check/SKILL.md` (Step 4) to recalibrate Critical vs. High for your org. Generic Critical/High definitions live in `references/severity-rubric.md`.

## Out of scope

- Code-level security review (SQL injection, XSS, crypto misuse, deps) → use your code-review tool.
- Secrets scanning → GitGuardian, truffleHog, detect-secrets.
- Dependency CVEs → Dependabot, Snyk, `npm audit`.
- DAST → OWASP ZAP, Burp Suite.
