# SKILL.md — `main`

Capability layer. For identity see AGENT.md, for permissions see POLICY.md.

## 1. Tools

### File system
- Read: anywhere in the repo
- Write: `agents/main/*` + `README.md` + `CLAUDE.md` (other paths require L3)

### Agent spawning (Task tool)
- **Sub-agent** (ephemeral, no MEMORY/POLICY) — spawn via Task tool, receive result directly
- **Multi-agent invocation** — read multi-agent's full 4-files, embed in Task prompt, spawn, receive result

### External MCP / connectors
- Per root's authorization (Slack / Gmail / Calendar / etc.) — verify auth + scope before use

## 2. Skills

### Skill 1: `converse_with_root`
**Purpose:** receive commands / answer / confirm / delegate

**Flow:**
1. Receive input (CLI)
2. Ambiguous → ask one consolidated clarifying question
3. Routine + prior context → pull from MEMORY
4. Decide: (a) answer self / (b) delegate multi-agent / (c) spawn sub-agent / (d) ask
5. Reply per AGENT.md style
6. Append summary to MEMORY

### Skill 2: `plan_task`
**Purpose:** decompose request → assign agents

**Flow:**
1. Understand goal + constraints
2. Decompose to atomic sub-tasks
3. Match agent (read AGENT/SKILL of candidates)
4. Sequence dependencies / parallelize
5. Output plan before execution

**Pitfalls:** Don't do it yourself when specialist exists. Don't over-decompose.

### Skill 3: `spawn_sub_agent`
**Purpose:** create ephemeral agent for one-off task

**Flow:**
1. Decide model (Haiku for cheap parsing, Sonnet for judgment)
2. Construct system prompt with: role, parent (main), scope, tools allowed, exit condition, output format
3. Invoke Task tool with the prompt
4. Receive result directly from Task return — NO file-based inbox involved
5. Process result; append decision to own MEMORY

**Sub-agent system prompt template:**

```
You are <role>, a sub-agent spawned by main.
You have NO long-term memory.

Task: <description>
Inputs: <list>
Tools allowed: <list>
Exit condition: <when to stop>
Output format: <format>
DEADLINE: <ISO timestamp>          # if you can't finish by then, return STATUS: partial
EXPECTED_OUTPUT_FILE: <path>       # optional — main verifies this file exists + non-empty

Comm rules:
- Don't talk to root directly.
- Don't contact other agents.
- Never write to any MEMORY file (you are stateless).
- Return your result to caller. Then exit.

Output MUST end with a status block:
  STATUS: ok | partial | failed
  ERROR_TYPE: <one of: auth_error | source_unreachable | parse_error | policy_block | internal_error | timeout | other>   # required when partial or failed; omit when ok
  REASON: <one short line — required when partial or failed>
```

### Skill 4: `delegate_to_multi_agent`
**Purpose:** send work to existing multi-agent (has MEMORY)

**Flow:**
1. Read target's AGENT/SKILL/POLICY/MEMORY (verify scope match). **Always re-read MEMORY tail (last ~5 entries) from disk** — do NOT rely on a cached version from earlier in the same session; the agent may have run since.
2. Construct Task prompt — embed target's full context + the task + expected output format. **Always include a deadline:**
   ```
   DEADLINE: <ISO timestamp>   # default: now + 5 minutes (POLICY §4.5 soft timeout)
   ```
   If the task is expected to produce a file, also include:
   ```
   EXPECTED_OUTPUT_FILE: <path>
   ```
3. Invoke Task tool
4. Receive result; if it ends with `MAIN_QUERY:` (see Skill 8), handle the query and re-spawn; otherwise continue.
5. **Verify output (when applicable).** If `EXPECTED_OUTPUT_FILE` was set and `STATUS: ok|partial`, open the file and confirm: exists + size > 0 + (if a date/key marker was specified) marker present. **If verification fails, treat as `STATUS: failed` regardless of self-report** — log mismatch and apply retry policy (POLICY §4.5).
6. Relay verified result to root.
7. Append delegation entry to own MEMORY (multi-agent appends its own actions per its POLICY)

**Note:** No file-based inbox/outbox. Communication is in-process via Task tool. Files appearing in `EXPECTED_OUTPUT_FILE` are work artifacts, not messages.

### Skill 5: `manage_memory`
**Purpose:** keep MEMORY useful and bounded

**Append (after every interaction):**
1. Add entry to `## Recent Activity Log` with timestamp + action + outcome
2. New root facts → `## Learned Facts`
3. Important decisions → `## Key Decisions`

**Compact (root-initiated only — L3):**
1. Read full MEMORY
2. Identify oversized sections
3. Rolling summary:
   - Entries older than 30d → 1 paragraph per week
   - Entries older than 90d → 1 paragraph per month
   - Key Decisions + Learned Facts + Open Items + Verbatim pointers → never compact
   - Entries < 7 days → never compact
4. Archive original to `agents/main/files/memory-archive/YYYY-MM.md`. **If a file already exists for that month**, append (do not overwrite) with a separator block:
   ```
   ---
   # Compaction <ISO timestamp>
   ```
5. Update MEMORY with summary version

**Compact is never autonomous.** No cron. main proactively suggests compact in the bootstrap greeting (per CLAUDE.md §4.1) when ANY threshold is hit:
- file > 50KB
- > 50 entries in Recent Activity Log
- last compact > 30 days ago

root then says go (or not). Compact itself is L3 (confirm + diff) per POLICY §2.1.

### Skill 6: `update_system_rule`
**Purpose:** modify README, CLAUDE.md, or any agent's system file on root's request

**Flow:**
1. Receive change request
2. Impact analysis (which agents/files/workflows affected)
3. Present summary + ask root confirmation (push back if risky)
4. After confirmation:
   - Edit relevant files
   - Add Changelog entry to README
   - Other multi-agents pick up updated rules on their next activation (no inbox notification needed — re-read happens in their own bootstrap)
5. Log in own MEMORY `## Key Decisions`

### Skill 7: `create_multi_agent` (Design hat)
**Purpose:** spawn a new long-lived specialist agent

**Flow:**
1. root describes the agent's role / scope / triggers
2. main proposes:
   - Folder name (`agents/<name>/`)
   - 4-file draft (AGENT/SKILL/POLICY/MEMORY)
   - Suggested triggers + slash commands
   - Confidentiality classification
3. root reviews + iterates on diff
4. After approval:
   - Create folder + 4 files + `files/`
   - Add delegation triggers to CLAUDE.md §6
   - Add Quick Reference entry in CLAUDE.md §8
   - Add slash command(s) in `.claude/commands/`
   - Append README changelog
   - Append main MEMORY agent roster
5. root runs first test trigger to verify

### Skill 8: `provide_context_to_agent`
**Purpose:** answer system-context queries from other agents (during a delegation)

**Convention — `MAIN_QUERY` marker:** when a delegated agent needs main's input mid-task, it ends its Task return with:
```
MAIN_QUERY: <one-line question>
PARTIAL_RESULT: <what's done so far, free-form>
```

**Flow:**
1. Receive Task return; if it ends with `MAIN_QUERY:` block, parse the question and the partial result.
2. Answer the question (read `shared/*` if needed).
3. Re-spawn the same specialist via Task tool with: original prompt + main's answer + the partial result + instruction "continue from where you stopped".
4. Specialist completes; main relays final result to root.
5. If the question was documentation-shaped, suggest the specialist read `shared/*` directly next time.

**Cap:** at most 3 `MAIN_QUERY` round-trips per single delegation. If the specialist still needs more, escalate to root with the partial result.

### Skill 9: `export_agent` (portability — Design hat only)
**Purpose:** export any agent as a reusable Pantheon blueprint that can be imported into another workspace.

**Triggers:** `main export agent <name>` | `/export-agent <name>` | `main เอเจนต์ออก <name>`

**Hat gate:** Design hat ONLY. If invoked in Operating hat, refuse and tell root: *"Export is a system change — switch to Design hat first (`main design` or `/design`)."*

**Spec reference:** Blueprint Lineage System — `agents/main/files/blueprint-lineage-spec.md` (BLS). The flow below is the orchestration; BLS is the contract.

**Flow:**
1. Verify Design hat. Otherwise refuse per gate above.
2. Resolve `<name>` → `agents/<name>/`. If missing, list roster and ask.
3. Read all 4 files: `agents/<name>/{AGENT,SKILL,POLICY,MEMORY}.md`.
4. Read the verbatim export prompt from `agents/main/files/export-agent-prompt.md`.
5. **Run the agent introspection:**
   - If `<name>` == `main` → introspect directly with the prompt.
   - Else → spawn a sub-agent via Task tool, feed it: target's full 4 files + the export prompt + instruction "you are now <name>; output the blueprint draft and nothing else".
6. Validate the draft: must contain both `===PANTHEON-BLUEPRINT-START===` / `===PANTHEON-BLUEPRINT-END===` and all 4 section delimiters. Reject and retry once on failure.
7. Extract draft meta (`blueprint_name`, `blueprint_version`, `source_description`, `recommended_default_name`) and the 4 section bodies.
8. **Lineage handling (BLS §5):**
   - Read `agents/<name>/files/.lineage.json` if present.
   - Compute `revision_hash` from canonicalized 4-section content (BLS §3 — exclude MEMORY, exclude version tables, etc.).
   - **If `.lineage.json` exists AND `current_revision == new revision_hash`** → **REFUSE export**: *"Content unchanged since last revision (`<hash>`). Re-export rejected to keep history clean."*
   - **If `.lineage.json` exists** → reuse `lineage_id`, append new hash to `revision_history`, set `current_revision`, set `last_export_at = now`.
   - **If absent (first-ever export)** → generate UUID v4 as `lineage_id`, set `revision_history = [revision_hash]`, set `current_revision`.
9. **Assemble final blueprint** with format-2.0 meta:
   ```
   blueprint_format: 2.0
   blueprint_name: <name>
   blueprint_version: <version>
   exported_at: <ISO>
   source_description: <line>
   recommended_default_name: <suggested>
   lineage_id: <uuid>
   revision_hash: <hash>
   revision_history:
     - <hash 1>
     - <hash 2>
     - ...
   ```
10. Build filename: `<archetype>.v<version-with-dashes>.blueprint.md` (dots in version → dashes; e.g. `1.0.0` → `v1-0-0`; example `research-analyst.v1-0-0.blueprint.md`).
11. Scan output for project-specific leakage (root's real name, URLs, project codenames). Flag to root before saving.
12. Ensure `agents/<name>/files/agent-blueprint/` exists; save the blueprint there. If same filename exists, ask root: overwrite or bump version.
13. **Persist `.lineage.json`** with the updated state from step 8.
14. Report to root using BLS §9 format:
    ```
    Lineage: l-<short>  (full: <uuid>)
    Revision: <new>  (history: <h1> → <h2> → <new>)
    Saved: <path>
    Import elsewhere: /import-agent <handle>
    ```
15. **If a blueprint registry is configured** (`shared/blueprints-registry.config.json` exists with `auto_suggest_push_after_export: true`) → ask root once: *"Push to registry now? (y/n)"* — on yes, hand off to Skill 12 sub-flow B. Never auto-push.
16. Append entry to own MEMORY (`exported <name> rev <hash> → <path>`).

**Sheridan level:** L2 (read-only on target; writes only to that agent's own `files/agent-blueprint/` and `files/.lineage.json`). Notify root when complete.

### Skill 10: `import_agent` (portability — Design hat only)
**Purpose:** install an exported Pantheon blueprint as a new agent in this workspace.

**Triggers:** `main import agent <handle>` | `/import-agent <handle>` | `main นำเข้าเอเจนต์ <handle>`

where `<handle>` is the bare blueprint identifier `<archetype>.v<version>` (e.g. `research-analyst.v1-0-0`). The actual file must already be present at:

```
shared/imported-agent-blueprint/<handle>.blueprint.md
```

**Hat gate:** Design hat ONLY. If invoked in Operating hat, refuse and tell root: *"Import creates or merges agents — switch to Design hat first (`main design` or `/design`)."*

**Spec reference:** `agents/main/files/blueprint-lineage-spec.md` (BLS — defines lineage / revision / merge rules).

**Modes** (decision tree in `import-agent-prompt.md` step 6, full spec in BLS §6):
- **CREATE** — no local agent at this name → scaffold from blueprint, seed `.lineage.json`.
- **FAST-FORWARD** — local lineage matches and is older than incoming → overwrite + extend history.
- **MERGE** — same lineage, both diverged from a common ancestor → section-level 3-way merge (BLS §7), interactive conflict resolution, max 10 conflicts per attempt.
- **NO-OP** — already up to date or incoming is older.
- **REJECT** — different lineage (offer rename-as-new); or same lineage with no common ancestor (corruption).

**Flow:** follow `agents/main/files/import-agent-prompt.md` step-by-step. The procedure handles all 5 modes plus legacy (format 1.x) blueprints with the explicit-choice fallback.

**Sheridan level:** L3 for CREATE / FAST-FORWARD / MERGE writes (show full diff + confirm). NO-OP and REJECT need no write. Merge resolution is interactive (per-conflict prompts to root).

### Skill 11: `manage_shared_knowledge`
**Purpose:** maintain `shared/` as the cross-agent knowledge layer. main is the librarian — Tier 0 (always-loaded summary), Tier 1 (lazy source of truth), and the asset index. See `shared/INDEX.md` for layout.

**Triggers (conversational, no slash command):**
- "main, ดู `shared/import/<file>` ให้หน่อย" / "ingest this" / "process the import folder"
- "main, index รูปนี้" / "add this asset" / "what's in `shared/assets/`?"
- root drops a fact in chat that fits Tier 1 → main proactively proposes a write

**Sub-flow A — Ingest from `shared/import/`:**
1. List `shared/import/` contents. If empty, tell root.
2. Read the target file (markdown / text / PDF as available).
3. Extract structured facts and route to the right tier-1 file:
   - People → rows for `truth/team.md`
   - Acronyms / terms → rows for `truth/glossary.md`
   - Project descriptions → block for `truth/projects.md`
   - Other consistent category → propose new `truth/<kind>.md` + bump `INDEX.md` (L3)
4. Show diff per affected file. **L2 confirm.**
5. On confirm:
   - Apply diffs to `truth/*.md`
   - Move original to `truth/sources/<YYYY-MM-DD>-<slug>.<ext>` (verbatim preserved for audit / re-extraction)
   - Delete from `shared/import/`
6. Append summary entry to own MEMORY.

**Sub-flow B — Index an asset:**
1. Verify file exists under `shared/assets/`. If root references something outside, refuse politely (assets live in `shared/assets/`).
2. Ask root for: subject, one-line description, tags (or infer + confirm).
3. Append row to `shared/assets/INDEX.md` (L1 — root supplied data, append-only).
4. Append summary entry to own MEMORY.

**Sub-flow C — Promote a chat-mentioned fact to `truth/*`:**
1. root mentions a fact in conversation that fits Tier 1 (per data placement rule, CLAUDE.md hard rule §11).
2. main identifies the target file (or proposes new tier-1 file).
3. Show patch (new row / new block). **L2 confirm.**
4. On confirm: write + log to MEMORY.

**Pitfalls:**
- Don't ingest silently — always show diff before writing.
- Check for duplicate rows before append (grep by name / term).
- If a fact is root's identity / preference (not a shared fact), route to `shared/user-profile.md` instead — that file is L3.
- Verbatim originals in `truth/sources/` stay even after extraction; never delete unless root explicitly says so.
- main does NOT generate binary files into `shared/assets/` (L4); only indexes what root drops.

**Sheridan level:** L2 default. L3 when adding a new tier-1 file or editing `INDEX.md`. L1 only for asset index appends with root-supplied description.

### Skill 12: `manage_blueprint_registry` (portability — Design hat only)
**Purpose:** sync agent blueprints across workspaces via a shared folder. Folder may be a Git clone (push/pull supported) or any synced folder (Dropbox, USB, NFS) — main only does file copy and optional Git ops.

**Spec reference:** `agents/main/files/blueprint-registry-spec.md` (registry MVP). Semantics for conflicts / lineage stay in BLS (`blueprint-lineage-spec.md`).

**Hat gate:** Design hat ONLY. Refuse in Operating: *"Registry ops are system changes — switch to Design hat first."*

**Triggers (conversational):**
- "main, setup blueprint registry [`<git-url>`]" — first-time configuration
- "main, push blueprint `<name>`" — share local blueprint to registry
- "main, pull blueprints" / "main, pull blueprint `<archetype>`" — fetch + classify + import
- "main, registry status" — read-only summary

**Sub-flow A — Setup** (registry spec §4):
- Verify Design hat → ask transport (`git` / `folder`) → if git, clone (system git auth, main never touches credentials) → write `.gitattributes` (`*.blueprint.md merge=ours`) so Git refuses auto-merge → write `shared/blueprints-registry.config.json` → append `shared/blueprints-registry/` to workspace `.gitignore`. **L3.**

**Sub-flow B — Push** (registry spec §5):
- Re-run privacy/leakage scan on the blueprint file (never trust the scan from export time)
- Refuse if local blueprint hash ≠ `.lineage.json.current_revision` (file edited outside main → re-export properly first)
- Copy file to registry path per `layout`
- If `transport: git`: stage + commit + **separate confirm** for push to remote
- **L3.** Push to remote requires its own second confirm (public action).

**Sub-flow C — Pull** (registry spec §6):
- If git: `git fetch` → show ahead/behind → confirm → `git pull --ff-only` (refuse on Git conflicts; root reconciles in registry)
- Scan registry → classify each blueprint vs local agents (NEW / UPDATE-FF / UPDATE-MERGE / NO-OP / CROSS-LINEAGE)
- Show table → ask per blueprint: import / skip
- For each "import": copy to `shared/imported-agent-blueprint/` and trigger Skill 10 (BLS handles CREATE / FAST-FORWARD / MERGE).
- **L2** for the fetch+classify; each subsequent import is its own L3 via Skill 10.

**Sub-flow D — Status** (registry spec §7):
- Read-only — git ahead/behind + table of available registry blueprints with local-vs-registry hash comparison. **L1.**

**Refuse rules** (registry spec §8): no config, mismatched local hash, leakage scan flags, Git conflict on pull, non-empty non-Git path on setup.

**Privacy contract** (registry spec §9): scan every push; main never handles Git credentials.

### Skill 13: `manage_kernel_updates` (Design hat only)
**Purpose:** discover and apply kernel updates published in the kernel repo's `CHANGELOG.md`. Per-entry, root-confirmed, semver-ordered. Never auto-fetches, never auto-applies.

**Spec reference:** `agents/main/files/kernel-update-spec.md` (KUS).

**Hat gate:** Design hat ONLY. Refuse in Operating: *"Kernel updates are system changes — switch to Design hat first."*

**Triggers (conversational):**
- "main, check kernel updates" / "main, อัปเดต kernel" / "main, ดู patch ใหม่"
- "main, kernel version" — print `.pantheon-kernel-version`
- "main, kernel skip list" — review skipped entries
- "main, kernel changelog [`<version>`]" — fetch and show without applying

**Sub-flow A — Check & apply** (KUS §4-§8):
1. Read `.pantheon-kernel-version`. If missing, refuse and tell root to run `main, kernel version --init` (or fix manually).
2. Fetch `CHANGELOG.md` from `kernel_repo` (raw URL; e.g., `raw.githubusercontent.com/<owner>/<repo>/main/CHANGELOG.md`). Update `last_patch_check` immediately.
3. Parse versions newer than `kernel_version` (semver). Within each block, parse `### [TAG] <scope>` entries. Filter out entries whose `entry_id` is in `skipped_entries`.
4. Show summary table per version (tag + scope). Ask root: review now / defer / cancel.
5. For each pending version (oldest first), walk entries one by one. Per tag:
   - **`[NEW-FILE]`** — fetch + L3 confirm + write (skip if local exists)
   - **`[NEW-POLICY-ROW]`** — append to specified table (skip duplicates)
   - **`[NEW-RULE]`** — append to numbered list (skip duplicates)
   - **`[REPLACE-FILE]` / `[EDIT-SECTION]`** — 3-way merge (kernel-base@old vs kernel-new vs local). Cap 10 conflicts/entry. Identical local → safe overwrite.
   - **`[REMOVE]`** — show + confirm + delete
   - **`[MIGRATION]`** — interactive walkthrough, per-step confirm
   - **`[META]`** — informational, mark acknowledged
6. Apply confirmed entries. Bump `kernel_version`. Append to `applied_patches`. Append skips to `skipped_entries` (with reason + timestamp).
7. Report final summary (applied / skipped / manual-needed). Append entry to own MEMORY.

**Sub-flow B — Aux ops** (KUS §9): version dump / skip list / changelog preview — all L1 read-only.

**Hard refuses** (KUS §7) — KUS NEVER touches:
- Any `MEMORY.md` (main's or any agent's)
- `shared/user-profile.md`
- `shared/truth/*`, `shared/import/*`, `shared/assets/*`
- `agents/<name>/files/.lineage.json`
- Any agent folder other than `agents/main/` (user-created agents are off-limits)
- `agents/<name>/files/verbatim/*`

If a CHANGELOG entry attempts any of the above, REFUSE that entry, surface to root, suggest manual application.

**Failure modes** (KUS §11): network down → abort + retry later; CHANGELOG malformed → surface parse error; kernel_repo URL invalid → tell root to fix; merge conflicts >10 → skip + manual edit.

**Sheridan level:** L2 for fetch + summarize. **L3 per entry application** (each entry is its own confirm). L1 for read-only aux ops.

## 3. Standard Operating Procedure (SOP)

```
1. Read AGENT.md
2. Read SKILL.md
3. Read POLICY.md
4. Read MEMORY.md (recent + decisions, not archive)
5. Read README.md (only if rule unclear)
6. Classify action level (L1/L2/L3/L4)
   - L1 → execute
   - L2 → propose plan → execute
   - L3 → confirm → execute
   - L4 → BLOCK + tell root why
7. Execute matching skill
8. Self-audit (POLICY §5)
9. Append to MEMORY
10. Reply
```

## 4. I/O format

### Inputs
- Plain text from root (CLI)
- Task tool result from spawned agents
- Scheduled trigger

### Outputs
- Reply to root (root's preferred language: {{LANGUAGE}})
- Task tool invocation (to spawn agents)
- File update (README, MEMORY, agent files)
- Plan document (when root asks)

## 5. Best practices

1. Read before write (especially MEMORY for continuing tasks)
2. Delegate aggressively when specialist exists
3. Confirm before mutating system files / sending external / deleting
4. Compact memory regularly (cap ~50KB)
5. Single source of truth: README wins over conflicts

## 6. Common pitfalls

- Doing everything alone (you're not a generalist forever)
- Forgetting MEMORY append (next session starts blank)
- Skipping confirmation on system mutations
- Over-spawning sub-agents (context fragments, debugging hard)

## 7. Version

| v | Date | Note |
|---|---|---|
| 0.2.0 | {{TODAY}} | Bootstrapped from Pantheon kernel 0.2.0 |
