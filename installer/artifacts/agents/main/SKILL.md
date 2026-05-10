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

**Flow:**
1. Verify Design hat. Otherwise refuse per gate above.
2. Resolve `<name>` → `agents/<name>/`. If missing, list roster and ask.
3. Read all 4 files: `agents/<name>/{AGENT,SKILL,POLICY,MEMORY}.md`.
4. Read the verbatim export prompt from `agents/main/files/export-agent-prompt.md`.
5. Run the export against the target agent:
   - If `<name>` == `main` → introspect directly with the prompt.
   - Else → spawn a sub-agent via Task tool, feed it: target's full 4 files + the export prompt + instruction "you are now <name>; output the blueprint and nothing else".
6. Validate the returned blueprint: must contain both `===PANTHEON-BLUEPRINT-START===` and `===PANTHEON-BLUEPRINT-END===` plus all 4 section delimiters. Reject and retry once on failure.
7. Parse the meta block to extract `blueprint_name` (the archetype, e.g. `research-analyst`) and `blueprint_version` (e.g. `1.0.0`).
8. Build the filename: `<archetype>.v<version-with-dashes>.blueprint.md`
   - Convert dots in version → dashes: `1.0.0` → `v1-0-0`
   - Example: `research-analyst.v1-0-0.blueprint.md`
9. Scan output for project-specific leakage (root's real name, addresses, URLs, project codenames). If found, flag to root before saving.
10. Ensure the per-agent blueprint folder exists: `agents/<name>/files/agent-blueprint/` (create if missing).
11. Save to `agents/<name>/files/agent-blueprint/<archetype>.v<version>.blueprint.md`. If a file with the same name exists, ask root: overwrite, or bump version.
12. Reply to root with the saved file path and the corresponding import command for the destination workspace:
    `/import-agent <archetype>.v<version>` (after dropping the file in `shared/imported-agent-blueprint/` there).
13. Append entry to own MEMORY (`exported <name> → <path>`).

**Sheridan level:** L2 (read-only on target; writes only to the target agent's own `files/agent-blueprint/`). Notify root when complete.

### Skill 10: `import_agent` (portability — Design hat only)
**Purpose:** install an exported Pantheon blueprint as a new agent in this workspace.

**Triggers:** `main import agent <handle>` | `/import-agent <handle>` | `main นำเข้าเอเจนต์ <handle>`

where `<handle>` is the bare blueprint identifier `<archetype>.v<version>` (e.g. `research-analyst.v1-0-0`). The actual file must already be present at:

```
shared/imported-agent-blueprint/<handle>.blueprint.md
```

**Hat gate:** Design hat ONLY. If invoked in Operating hat, refuse and tell root: *"Import creates a new agent — switch to Design hat first (`main design` or `/design`)."*

**Flow:** follow the procedure in `agents/main/files/import-agent-prompt.md` step-by-step.

**Sheridan level:** L3 (creates new agent folder + edits CLAUDE.md / README — show full diff and require root confirm before writing).

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
