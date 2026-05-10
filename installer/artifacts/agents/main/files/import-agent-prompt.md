# Import-Agent Procedure

> Used by `main` Skill 10 (`import_agent`). Run this when root provides a
> Pantheon blueprint handle. **Design hat only.**
>
> **Authoritative spec for lineage / merge:** see
> `agents/main/files/blueprint-lineage-spec.md` (BLS). This procedure is the
> orchestration; the spec is the contract. Always defer to the spec on
> ambiguous cases.

## Convention

- Blueprint file location (fixed): `shared/imported-agent-blueprint/<handle>.blueprint.md`
- Handle format: `<archetype>.v<version-with-dashes>` — e.g. `research-analyst.v1-0-0`
- Invocation: `/import-agent <handle>` (without `.blueprint.md` suffix and without
  the directory prefix — both are implicit).

If `shared/imported-agent-blueprint/` does not exist, create it before checking
for the file (so the next import has a clear drop point).

## Steps

1. **Verify Design hat.** If Operating, refuse: *"Import creates or merges agents — switch to Design hat first (`main design` or `/design`)."*

2. **Resolve the blueprint path** from `$ARGUMENTS`:
   - Accept bare handle: `research-analyst.v1-0-0` → `shared/imported-agent-blueprint/research-analyst.v1-0-0.blueprint.md`.
   - Accept handle with `.blueprint.md`: strip and resolve as above.
   - If `$ARGUMENTS` is empty → list every `*.blueprint.md` in `shared/imported-agent-blueprint/` and ask. If empty, instruct root to drop the file there first.
   - Reject absolute paths or anything outside `shared/imported-agent-blueprint/`.

3. **Read and validate the blueprint.** Must contain both `===PANTHEON-BLUEPRINT-START===` and `===PANTHEON-BLUEPRINT-END===` plus the four section delimiters. Reject otherwise.

4. **Parse the meta block.** Extract: `blueprint_format` (default `1.0` if missing), `blueprint_name`, `blueprint_version`, `recommended_default_name`, `source_description`, and (if `blueprint_format >= 2.0`) `lineage_id`, `revision_hash`, `revision_history`. Show a short summary to root including the lineage short id and revision hash if present.

5. **Pick the target agent name.** Default to `recommended_default_name`; ask root for an override if they want one.

6. **Branch on `blueprint_format` and local agent state — follow the BLS decision tree (`blueprint-lineage-spec.md` §6).** Summary:

   | Local state | Incoming format | Outcome |
   |---|---|---|
   | No agent at this name | any | **CREATE** (step 7a) |
   | Agent exists, no `.lineage.json` | 1.x (legacy) | Ask: rename / overwrite / abort |
   | Agent exists, no `.lineage.json` | 2.0 | Ask: rename / adopt-lineage-and-overwrite / abort |
   | Agent exists, has `.lineage.json`, lineage_id mismatch | 2.0 | **REJECT** — offer rename-as-new |
   | Agent exists, lineage_id match, incoming hash already in local history | 2.0 | NO-OP |
   | Agent exists, lineage_id match, local hash in incoming history (newer) | 2.0 | **FAST-FORWARD** (step 7b) |
   | Agent exists, lineage_id match, diverged with common ancestor | 2.0 | **MERGE** (step 7c) |
   | Agent exists, lineage_id match, no common ancestor in histories | 2.0 | REJECT (corruption — surface to root) |

7. **Apply the chosen mode:**

   **7a. CREATE**
   - Collect placeholder values: `{{AGENT_NAME}}`, `{{ROOT_NAME}}`, `{{ROOT_ADDRESS}}`, `{{ROOT_PRONOUN}}`, `{{ROOT_LANGUAGE}}` (from `shared/user-profile.md`), `{{TODAY}}`, `{{PROJECT_CONTEXT}}` (ask root for one line, or pull from `shared/conventions.md`).
   - Split blueprint into 4 sections; substitute placeholders.
   - Show diff. **L3 confirm.**
   - Write `agents/<name>/{AGENT,SKILL,POLICY,MEMORY}.md` + `agents/<name>/files/.gitkeep`.
   - Write `agents/<name>/files/.lineage.json`:
     - If incoming had lineage (format 2.0) → copy `lineage_id`, set `current_revision = incoming.revision_hash`, `revision_history = incoming.revision_history`.
     - If incoming was legacy (format 1.x) → generate a new UUID v4 as `lineage_id`, compute `current_revision` from the freshly written content per BLS §3, set `revision_history = [current_revision]`.
   - Set `imported_from_blueprint = <handle>.blueprint.md`, `last_import_at = now`, `last_export_at = null`.

   **7b. FAST-FORWARD**
   - Substitute placeholders in incoming sections.
   - Show diff (local content vs incoming). **L3 confirm with explicit warning** that local edits since the agent's last revision will be overwritten — if local has uncommitted changes that aren't reflected in `current_revision`, those will be LOST. Suggest exporting locally first to capture them as a divergent revision.
   - Overwrite the 4 files.
   - Update `.lineage.json`: `current_revision = incoming.revision_hash`, `revision_history = incoming.revision_history` (which already contains the local one), `last_import_at = now`, `imported_from_blueprint = <handle>.blueprint.md`.

   **7c. MERGE** — follow BLS §7 (section-level 3-way merge):
   1. Compute the common ancestor — the latest hash present in BOTH `local.revision_history` and `incoming.revision_history`. If found nowhere, REJECT.
   2. **Reconstruct ancestor content.** This is the hard part: ancestor content is not stored locally — only its hash. Two paths:
      - **Best:** if root has the ancestor blueprint file in `shared/imported-agent-blueprint/` (any older import), read it for ancestor content.
      - **Fallback:** ask root to drop the ancestor blueprint into `shared/imported-agent-blueprint/<archetype>.<ancestor-handle>.blueprint.md` before continuing.
      - **Degraded:** if ancestor unavailable, offer 2-way merge instead (local vs incoming, no ancestor) — every differing section becomes a CONFLICT (no auto-merge possible). Warn root before proceeding.
   3. Parse local + incoming + ancestor into sections (top-level `## N. Title` for AGENT/POLICY; per-skill `### Skill N: name` for SKILL).
   4. Classify each section per BLS §7 step 2 (no-change / only-local / only-incoming / both-changed = CONFLICT / etc.).
   5. **Cap at 10 conflicts.** If exceeded, abort with suggestion to merge in smaller doses (e.g., revert one side first, merge piecewise).
   6. For each CONFLICT, present to root verbatim:
      ```
      CONFLICT — <file> §<section title>
      --- ancestor (revision <ancestor-hash>) ---
      <text>
      --- local (revision <local-current>) ---
      <text>
      --- incoming (revision <incoming-current>) ---
      <text>
      Choose: (a) keep local (b) take incoming (c) edit manually (paste new content) (d) abort merge
      ```
   7. After resolving every conflict, assemble merged 4 files.
   8. Compute new `revision_hash` from canonicalized merged content (BLS §3).
   9. Build new `revision_history` = ordered union of local + incoming histories (chronological by appearance), then append the new merge hash.
   10. Show final summary (sections changed / conflicts resolved / new hash) → **L3 confirm**.
   11. Write merged 4 files.
   12. Update `.lineage.json`: `current_revision = new merge hash`, `revision_history = merged + new`, `last_import_at = now`, `imported_from_blueprint = <handle>.blueprint.md`.

8. **Register the agent (CREATE path only):**
   - Add row to main MEMORY §6 Agent roster.
   - Add delegation triggers / Quick Reference entry to `CLAUDE.md` if root wants.
   - Append README changelog: `imported <blueprint_name>@<blueprint_version> as <name>`.

9. **Report to root** with the BLS report format (BLS §9):
   ```
   Lineage: l-<short>  (full: <full-uuid>)
   Revision: <new-current>  (history: <hash> → <hash> → <new-current>)
   Mode: CREATE | FAST-FORWARD | MERGE
   Saved: agents/<name>/
   ```
   Then suggest `/connect <name>` for a test run.

## Refuse if
- Invoked outside Design hat.
- Blueprint file not found at `shared/imported-agent-blueprint/<handle>.blueprint.md`.
- Blueprint delimiters missing or mismatched.
- Blueprint contains obvious project-specific leakage (specific person names, URLs, dates) — flag to root and ask whether to clean before importing.
- Lineage mismatch with existing local agent (different `lineage_id`) — offer rename-as-new instead.
- Same lineage but no common ancestor found in histories — surface as corruption.
- More than 10 conflicts in a single merge attempt.
