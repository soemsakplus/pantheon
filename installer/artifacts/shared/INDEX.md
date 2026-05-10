# `shared/` — Knowledge index

> Catalog of cross-agent shared knowledge. main is the librarian. Every agent reads this on bootstrap to know what's available.

## Tier 0 — always loaded (small, every bootstrap)

| File | Purpose | Source of truth for |
|---|---|---|
| `INDEX.md` | this catalog | structure of `shared/` |
| `user-profile.md` | root identity + comm preferences | who root is, how to address, language |
| `conventions.md` | cross-cutting project rules | file naming, dates, markdown style, git, inter-agent comm |

## Tier 1 — lazy-loaded source of truth (load when relevant)

| File | Purpose | Read by |
|---|---|---|
| `truth/team.md` | team roster (name, role, contact, timezone) | any agent doing person lookups |
| `truth/glossary.md` | acronyms, codenames, internal terms | any agent encountering an unfamiliar term |
| `truth/projects.md` | active project context (one-paragraph each) | planning, reporting, status agents |
| `truth/sources/` | verbatim originals after ingest (re-reference / audit) | main on demand |

## Inbox & assets

| Folder | Purpose | Workflow |
|---|---|---|
| `import/` | drop zone for text/doc files root wants main to ingest | root drops → "main, ดู `shared/import/<file>` ให้หน่อย" → main extracts → propose patch to `truth/*` → confirm → write + (optionally) move original to `truth/sources/` → delete from `import/` |
| `assets/` | binary assets (photos, logos, diagrams) for tools like slide gen | root drops file manually → "main, index รูปนี้" → main appends row to `assets/INDEX.md` |
| `imported-agent-blueprint/` | drop zone for `/import-agent` blueprints | see [README.md](../README.md) "Export / Import agents" |

## Adding a new tier-1 file

When repeated facts of a new kind start appearing (e.g. vendors, OKRs, customers), main proposes adding a new `truth/<kind>.md`. Bumps this INDEX. **L3** — root confirms.

## Data placement rule (CLAUDE.md hard rule §11)

| Lifetime / scope | Lives in |
|---|---|
| One agent's experience (action log, decisions) | `agents/<name>/MEMORY.md` |
| Root identity / preferences | `shared/user-profile.md` |
| Facts multiple agents need | `shared/truth/*.md` |
| Verbatim originals after ingest | `shared/truth/sources/` |
| Binary assets | `shared/assets/` |

When unsure: **"does another agent need this?"** — yes = `shared/`, no = `MEMORY`.

## Version

| v | Date | Note |
|---|---|---|
| 0.1.0 | {{TODAY}} | Initial catalog (Pantheon kernel 0.3.0) |
