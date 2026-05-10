# Kernel Update System (KUS) — spec

> Loaded by main during Skill 13 (`manage_kernel_updates`) flows. Defines how an installed pantheon discovers and applies kernel updates published in the kernel repo's `CHANGELOG.md`.

## 1. Concept

When the kernel author releases a new version, the kernel repo's `CHANGELOG.md` gets a new section with structured entries (one per change, tagged: `[NEW-FILE]`, `[NEW-POLICY-ROW]`, `[NEW-RULE]`, `[REPLACE-FILE]`, `[EDIT-SECTION]`, `[REMOVE]`, `[MIGRATION]`, `[META]`).

main fetches the CHANGELOG, identifies versions newer than what this workspace runs, walks root through each entry, and applies what root chooses. **Nothing auto-applies. Nothing auto-fetches.** All updates are root-triggered, L3-confirmed, and per-entry.

## 2. Workspace version state — `.pantheon-kernel-version`

Path: workspace root. JSON.

```json
{
  "kernel_version": "0.3.1",
  "kernel_repo": "https://github.com/soemsakplus/pantheon",
  "installed_at": "2026-05-10T08:00:00Z",
  "last_patch_check": null,
  "applied_patches": ["0.3.1"],
  "skipped_entries": []
}
```

Field rules:
- `kernel_version` — the highest version that has been **fully applied** to this workspace.
- `kernel_repo` — canonical kernel repo URL. Editable: if root cloned from a fork, they replace this with the fork URL.
- `installed_at` — ISO timestamp; written once at install (never edited after).
- `last_patch_check` — ISO timestamp of the most recent `main, check kernel updates`; updated on every check (whether or not anything was applied).
- `applied_patches` — list of versions ever applied (just version strings; for audit).
- `skipped_entries` — list of `{version, entry_id, reason}` so main does not nag root about the same skipped change repeatedly.

`entry_id` = stable id derived from the entry: hash of `version + tag + first-line-text`, truncated 8 hex. Computed at apply time.

## 3. Trigger rules

- **Only root-triggered.** Bootstrap NEVER auto-checks (no surprise network calls, no slowed greeting).
- Triggers (conversational, no slash command):
  - "main, check kernel updates"
  - "main, อัปเดต kernel"
  - "main, ดู patch ใหม่"
- Re-runs are idempotent: nothing applied twice; skipped entries stay skipped unless root resets.

## 4. Fetch & parse

1. Read `.pantheon-kernel-version` → get `kernel_repo`, `kernel_version`, `skipped_entries`.
2. Compose raw URL for CHANGELOG. For GitHub: `https://raw.githubusercontent.com/<owner>/<repo>/main/CHANGELOG.md`. (For other forges, derive accordingly; prompt root if unsure.)
3. Fetch via WebFetch tool (or curl). On network failure: surface stderr + abort.
4. Parse CHANGELOG into version blocks (each `## X.Y.Z — YYYY-MM-DD` heading starts a block).
5. Filter to versions strictly greater than `kernel_version` (semver compare).
6. Within each new version block, parse each `### [TAG] <scope>` entry. Compute `entry_id` for each. Skip entries whose `entry_id` is already in `skipped_entries`.
7. Update `last_patch_check` immediately (regardless of what root does next).

## 5. Report to root

For each new version, show a compact summary:

```
Kernel 0.3.1 (2026-06-01) — 5 entries:
  [NEW-FILE]      shared/private/.gitkeep
  [NEW-POLICY]    POLICY.md §2.1 row for shared/private
  [NEW-RULE]      CLAUDE.md §12 PII guard
  [REPLACE-FILE]  agents/main/SKILL.md (Skill 9 leakage scan)
  [MIGRATION]     none

Already skipped: 0
Review now? (y / n / details)
```

If multiple versions are pending, suggest applying in order (oldest first) — refuse to skip versions out of order (semver ordering preserves migration dependencies).

## 6. Apply per entry

For each entry root chose to review, classify by tag:

### `[NEW-FILE] <path>`
- Check if local file exists.
  - **Absent** → fetch the file content from kernel repo at the new version's tag → show preview → L3 confirm → write.
  - **Present** → SKIP with reason "already exists locally" (don't overwrite). Add to `skipped_entries` with reason.

### `[NEW-POLICY-ROW] <path> §<section>`
- Read local file. Parse the table at `<section>`.
- Show the row to be appended.
- Detect duplicates (same first column already present) → SKIP with reason.
- Otherwise → L3 confirm → append row.

### `[NEW-RULE] <path> §<section>`
- Read local rule list. Detect duplicate rule (same first ~15 words) → SKIP.
- Otherwise → L3 confirm → append at the end of the list with the next number.

### `[REPLACE-FILE] <path>` and `[EDIT-SECTION] <path> §<section>`
- 3-way merge required (kernel-base@old vs kernel-new vs local-current).
- Fetch kernel-base from kernel repo at tag of `kernel_version` (the user's currently-applied version).
- Fetch kernel-new from kernel repo at tag of the new version.
- Compare local vs kernel-base:
  - **Identical** → safe overwrite with kernel-new (L3 confirm with diff).
  - **Diverged** → 3-way merge. Same engine as BLS §7 (section-level for markdown; line-level only as fallback). Cap 10 conflicts/entry.
- If user can't resolve, offer SKIP + add to `skipped_entries` with reason "manual merge needed".

### `[REMOVE] <path>` or `[REMOVE] <path> §<section>`
- Show what would be removed + the reason from the changelog entry.
- L3 confirm. After removal, also offer to remove orphan references from CLAUDE.md / README if any.

### `[MIGRATION]`
- The entry body contains step-by-step instructions in script form (`mv X Y`, `update field Z`, etc.).
- Walk root through each step interactively. Apply with confirm per step.
- If migration needs data root must supply (e.g., new config field), ask explicitly.

### `[META]`
- Informational only. Show to root. No action. Mark as acknowledged (still recorded in applied entries, not skipped).

## 7. Hard refuses (NEVER touch via kernel update)

- `agents/main/MEMORY.md` and any `agents/<name>/MEMORY.md`
- `shared/user-profile.md`
- `shared/truth/*`, `shared/truth/sources/*`
- `shared/import/*`, `shared/assets/*`
- Any `agents/<name>/files/.lineage.json`
- Any agent folder that is NOT `agents/main/` (i.e., user-created multi-agents)
- Verbatim files at `agents/<name>/files/verbatim/*`

If a CHANGELOG entry attempts to touch any of the above paths, REFUSE the entry and surface to root: *"This patch wants to modify a path I refuse to touch (per KUS §7). Skipping. If you want this change, apply it manually."*

## 8. After all entries processed

1. If at least one entry was applied (or the version had zero entries needing action), update `kernel_version` to that version. Append the version string to `applied_patches`.
2. If any entries were skipped, append them to `skipped_entries` with `{version, entry_id, reason, decided_at}`.
3. Move to the next pending version (apply in order — oldest first).
4. After all pending versions processed, report:
   ```
   Kernel updated: 0.3.1 → 0.4.0
   Applied: 12 entries
   Skipped: 2 entries (review with `main, kernel skip list`)
   Manual edits required: 1 entry — see notes
   ```
5. Append summary entry to own MEMORY (`patched kernel 0.3.1 → 0.4.0`).

## 9. Aux operations & slash commands

**Slash commands (Skill 13 entry points):**
- **`/update-status`** ≡ "main, kernel update status" — fetch + summarize, **read-only**. No walkthrough, no apply prompts. Operating hat OK.
- **`/update`** ≡ "main, check kernel updates" — full interactive walkthrough + apply. Design hat required.

**Conversational aux:**
- **`main, kernel version`** — print current `.pantheon-kernel-version` contents.
- **`main, kernel skip list`** — list `skipped_entries` with reasons; offer to revisit each.
- **`main, kernel changelog [<version>]`** — fetch and show CHANGELOG (or one version block) without applying.
- **No revert.** If root regrets a patch, they use Git on the workspace itself (`git revert`, `git checkout`). KUS does not implement revert.

## 10. Rate / cost

- Network: one WebFetch per `check kernel updates` (just CHANGELOG). Per-file fetches happen only for entries root chose to apply.
- Kernel author publishes patches infrequently (Pantheon is a small kernel). Daily/weekly check is fine; bootstrap should never auto-check.

## 11. Failure modes

- **Network down** → abort with message; root retries later.
- **CHANGELOG malformed** → surface parse error + version + entry context; do nothing destructive.
- **Kernel repo URL invalid** → tell root to fix `kernel_repo` in `.pantheon-kernel-version`.
- **3-way merge has > 10 conflicts** → SKIP that entry; suggest manual edit.
- **Patch entry violates §7 refuse list** → SKIP + surface refusal reason.
