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

## 0.3.1 — 2026-05-10

Patch — fixes from the MVP completeness audit.

### [NEW-FILE] installer/artifacts/shared/assets/.gitkeep

Adds `.gitkeep` to the binary asset drop folder, consistent with the
pattern already used by `shared/import/` and `shared/truth/sources/`.

### [EDIT-SECTION] installer/INSTALL.md §Phase 5

Expands the post-install state tree to show the complete expected
repo layout: `.pantheon-kernel-version`, all 13 slash commands,
`agents/main/files/` spec docs, and the full `shared/` subfolder
structure (`truth/`, `import/`, `assets/`, `imported-agent-blueprint/`).

### [MIGRATION] None required

---

## 0.3.0 — 2026-05-10

KUS (Kernel Update System) ships. Installed pantheons can now patch toward future kernel versions via `/update-status` and `/update`.

### [META] First version with patchable upgrades

Pantheons installed at **0.2.0** do NOT have Skill 13 (`manage_kernel_updates`) and therefore cannot use `/update` to reach 0.3.0. The 0.2.0 → 0.3.0 jump is **bootstrap-only** and must be applied manually (copy the new files, install Skill 13, write `.pantheon-kernel-version`). From 0.3.0 onward, future versions are reachable via `/update`.

### [NEW-FILE] CHANGELOG.md (kernel root)

Strict structured changelog read by Skill 13. Each new release appends a `## X.Y.Z — date` block with tagged entries. Tag definitions in this file's header.

### [NEW-FILE] .pantheon-kernel-version (workspace root)

JSON state file tracking installed kernel version, kernel repo URL, applied patches, and skipped entries. Written by the installer at Phase 4.4.

### [NEW-FILE] installer/artifacts/agents/main/files/kernel-update-spec.md

Authoritative KUS contract: workspace state file, fetch/parse flow, per-tag apply behavior, hard refuse list (paths KUS NEVER touches), failure modes.

### [NEW-FILE] installer/artifacts/.claude/commands/update-status.md
### [NEW-FILE] installer/artifacts/.claude/commands/update.md

Slash commands wiring `/update-status` → "main, kernel update status" and `/update` → "main, check kernel updates".

### [EDIT-SECTION] installer/artifacts/agents/main/SKILL.md §Skill 13

Adds new Skill 13 `manage_kernel_updates` with three sub-flows:
- A — Status (read-only, Operating OK, L1)
- B — Check & apply (Design hat, L3 per entry)
- C — Aux ops (version dump / skip list / changelog preview, L1)

### [NEW-POLICY-ROW] installer/artifacts/agents/main/POLICY.md §2.1

Adds 9 rows covering KUS operations:

| Action | Level |
|---|---|
| Read `.pantheon-kernel-version` | L1 |
| Update `last_patch_check` | L1 |
| Update `kernel_version` / `applied_patches` / `skipped_entries` | L2 |
| Fetch CHANGELOG / kernel files (network) | L2 (Design hat, root-triggered only) |
| Apply `[NEW-FILE]` | L3 (Design hat) |
| Apply `[NEW-POLICY-ROW]` / `[NEW-RULE]` | L3 |
| Apply `[REPLACE-FILE]` / `[EDIT-SECTION]` | L3 (3-way merge) |
| Apply `[REMOVE]` | L3 |
| Apply `[MIGRATION]` | L3 (per-step confirm) |
| Touch refuse-list path via patch | L4 (forbidden) |

### [EDIT-SECTION] installer/artifacts/README.md §Kernel updates

New plain-English section: trigger commands, tag table, refuse list, no auto-check, no revert built-in, fork support.

### [EDIT-SECTION] installer/INSTALL.md §Phase 4.4

New install step: write `.pantheon-kernel-version` JSON to workspace root.

### [MIGRATION] None required for fresh installs

All changes are additive. Fresh installs at 0.3.0 work end-to-end. Existing 0.2.0 installs need the manual bootstrap noted in [META] above.

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
