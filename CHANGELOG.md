# Pantheon kernel — CHANGELOG

> Machine-readable changelog. Read by `main` Skill 13 (`manage_kernel_updates`)
> in installed pantheons during `main, check kernel updates`. Format is **strict** —
> see "Entry tags" below. Never reword tags. Never insert free prose between entries.

## Entry tags

Every change inside a version block must be one of:

| Tag | Meaning | Auto-applicable? |
|---|---|---|
| `[NEW-FILE]` | A new file/folder is added that did not previously exist | Yes if local file absent |
| `[NEW-POLICY-ROW]` | One row added to a POLICY.md table | Yes (append-only) |
| `[NEW-RULE]` | One new entry in a numbered hard-rule list | Yes (append-only) |
| `[REPLACE-FILE]` | An existing kernel file is rewritten | No — 3-way merge with user's local |
| `[EDIT-SECTION]` | A specific section inside a kernel file is rewritten | No — section-level merge |
| `[REMOVE]` | A file/section is removed | No — explicit confirm + reason |
| `[MIGRATION]` | Data movement / restructure required (script-form steps embedded) | No — manual walkthrough |
| `[META]` | Documentation/spec note that doesn't change code | Yes (informational) |

Each entry is a single `### [TAG] <path or scope>` heading followed by 1-3
short paragraphs. Use a fenced code block when proposing exact text to add.

## Versioning

Semver. Bump kernel version in `installer/artifacts/CLAUDE.md` §9, root
`README.md`, and add a new section here.

---

## 0.2.0 — 2026-05-10

Baseline. Includes:

- 4-file agent architecture (AGENT/SKILL/POLICY/MEMORY)
- Sheridan L1-L4 framework
- Two-hat main (Operating / Design)
- Direct Mode + state file invariants
- 11 hard rules (single writer per MEMORY, data placement, etc.)
- M1-M13 protocol tightening (verification, deadlines, error taxonomy, MAIN_QUERY)
- Shared knowledge layer (`shared/INDEX.md`, `shared/truth/`, `shared/import/`, `shared/assets/`)
- Blueprint Lineage System (BLS) — lineage_id + revision_history + 5-mode import
- Blueprint registry — git or folder transport, push/pull/status
- 12 main skills

Pre-0.2.0 history is summarized in `PANTHEON-INSTALL.md`.

---

<!--
Future template:

## 0.3.0 — YYYY-MM-DD

### [NEW-FILE] shared/private/.gitkeep
Adds private/ folder for the PII tier (export-safe sensitive data).
Local action: create the folder if missing.

### [NEW-POLICY-ROW] agents/main/POLICY.md §2.1
Append:

| Action | Level |
|---|---|
| Write `shared/private/*` | L3 (PII; double-confirm + warn export impact) |

### [NEW-RULE] CLAUDE.md hard rule §12
Append:

12. **PII isolation** — never include `shared/private/*` content in Task prompts to delegated agents (unless that agent's POLICY explicitly allows). Never include in export blueprints (Skill 9 must scan and reject).

### [EDIT-SECTION] agents/main/SKILL.md Skill 9 step 11
Replace the leakage scan step with a stricter PII tier check (see kernel commit for exact text).

### [MIGRATION] None required
-->
