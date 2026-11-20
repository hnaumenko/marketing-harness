# Data Contract

## User Layer (NEVER auto-updated)

| File | Purpose |
|---|---|
| `modes/_profile.md` | Team and project narrative: who we are, stakeholders, constraints |
| `modes/_custom.md` | House rules (procedural only, never factual claims) |
| `config/*` | Project name, output language |
| `docs/*` | Source documents, research, `assumptions.md` register, `decisions/` (ADRs), diagrams, specs, plans |
| `README.md` | Project deliverable (generated output); follows the same marker rules |

## System Layer (safe to auto-update)

| File | Purpose |
|---|---|
| `modes/_shared.md` | Behavioral core and global rules |
| `modes/*.md` (except `_profile.md`, `_custom.md`) | Task modes and templates |
| `AGENTS.md`, `CLAUDE.md`, `DATA_CONTRACT.md` | Agent instructions |
| `.claude/skills/*` | Skill router |

## The Rule

If a file is in the User Layer, no update process may read, modify, or delete it. If it is in the System Layer, it can be safely replaced with the latest version. System-layer files never contain project facts.
