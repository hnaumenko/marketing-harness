---
name: mdt
description: >-
  Guardrailed design-and-delivery workflow for the team. Use when the user
  wants an RFC or ADR, a risk check of AI-generated or cross-team code,
  confirmation of assumptions and sources, triage of incoming requests, a
  stakeholder message draft, or a review of team-value metrics.
arguments: mode
user_invocable: true
argument-hint: "[rfc | adr | assumptions | review-ai-code | intake | stakeholder-msg | metrics]"
license: MIT
---

# mdt — Router

## Mode Routing

Determine the mode from `$mode`:

| Input | Mode |
|---|---|
| (empty / no args) | `discovery` — show command menu |
| `rfc` | `rfc` |
| `adr` | `adr` |
| `assumptions` | `assumptions` |
| `review-ai-code` | `review-ai-code` |
| `intake` | `intake` |
| `stakeholder-msg` | `stakeholder-msg` |
| `metrics` | `metrics` |

If `$mode` does not match a known mode, show discovery.

## Discovery Mode (no arguments)

Show:

```
mdt — Command Center

Developers
  /mdt rfc              → draft a review-ready design document
  /mdt adr              → record one decision
  /mdt assumptions      → confirm and log claims, figures, sources
  /mdt review-ai-code   → risk-tier a change (AI-generated or another team's repo)

Managers
  /mdt intake           → triage new requests and propose an order
  /mdt stakeholder-msg  → draft a message to stakeholders (never sends)
  /mdt metrics          → review whether the team delivers value
```

## Context Loading by Mode

After determining the mode, load files with explicit Read calls before executing.

If `modes/_custom.md` exists, read it after `modes/_profile.md` and before the mode file. It holds house rules and procedural preferences; it may override workflow/style defaults but never adds factual claims.

### Modes that require the behavioral core

Read `modes/_shared.md` + `modes/_profile.md` (if exists) + `modes/_custom.md` (if exists) + `modes/{mode}.md`.

Applies to: `rfc`, `adr`, `assumptions`, `review-ai-code`, `intake`, `stakeholder-msg`.

### Standalone modes (lighter context)

Read `modes/_profile.md` (if exists) + `modes/_custom.md` (if exists) + `modes/{mode}.md`.

Applies to: `metrics`.

### Modes delegated to a subagent

Inject file **contents** into the worker's prompt rather than relying on it to read files:

```python
Agent(
    subagent_type="general-purpose",
    prompt="[content of modes/_shared.md]\n\n[content of modes/_profile.md if exists]\n\n[content of modes/_custom.md if exists]\n\n[content of modes/{mode}.md]\n\n[invocation-specific data]",
    description="mdt {mode}"
)
```

A subagent spawned this way is a **single-pass worker**: it MUST NOT spawn further subagents or invoke open-ended research or skill chains. Research stays bounded and inline.

Execute the instructions from the loaded mode file.
