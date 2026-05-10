# {{ADDRESS}}'s Pantheon

Multi-agent virtual office built on the Pantheon kernel.

**Bootstrapped:** {{TODAY}}
**Kernel:** 0.3.1
**Root:** {{ADDRESS}}
**Default language:** {{LANGUAGE}}

## What lives here

- **`main`** — chief of staff (Operating hat + Design hat)
- (your other agents will be listed here as you grow them)

## How to use

1. Open Claude Code in this repo
2. Type `main start` (or `/main`) to activate Operating hat
3. Type `main design` (or `/design`) to switch to Design hat
4. Type `main status` (or `/status`) to check current hat
5. Type `main เริ่ม <agent>` (or `/connect <agent>`) to open Direct Mode with a multi-agent

## Concept

- **root** = you (the human) — never an AI persona
- **main** = your chief of staff — the only AI you talk to by default. Has two hats:
  - **Operating hat** — runs work, delegates, manages day-to-day
  - **Design hat** — helps you design/edit the system itself
- **multi-agents** = specialists with their own MEMORY, spawned by main
- **sub-agents** = ephemeral workers (no memory), spawned via Task tool for one-off jobs

## How it works (plain English)

### The four files per agent
Every agent is described by 4 files. Read them in this order to understand any agent:
- **AGENT.md** — who I am (identity, role, personality)
- **SKILL.md** — what I can do (tools and step-by-step procedures)
- **POLICY.md** — what I'm allowed to do (permission levels for every action)
- **MEMORY.md** — what I've done (experience log, decisions, facts learned)

### Sheridan levels (L1–L4) — when does main ask permission?

Every action gets a level. The riskier or more irreversible, the higher the level.

| Level | What it means | Example |
|---|---|---|
| **L1** | Do it, then tell you | Append to own MEMORY, normal conversation |
| **L2** | Propose a plan first | Multi-agent delegation, ingest from `shared/import/` |
| **L3** | Explicit confirmation required (show diff) | Edit a system file, compact MEMORY, import an agent |
| **L4** | Forbidden — you do it yourself | Send email/Slack as you, delete an agent, write another agent's MEMORY |

main states the level briefly when it's L2 or higher.

### How memory works

Each agent keeps its own MEMORY — like a personal diary of what happened, what was decided, and what it learned about you. Three things to know:

1. **Each agent owns its own MEMORY.** Nobody edits another agent's MEMORY (not even main). When main delegates work to a specialist, both keep parallel records — main logs the delegation, the specialist logs the work.
2. **Only meaningful stuff is logged.** Not every clarifying question. Just L2+ actions, key decisions, completed tasks, new facts about you.
3. **No autonomous cleanup.** When MEMORY grows (>50KB, >50 entries, or last compaction >30 days ago), main flags it in the morning greeting. You say go; main shows the diff and compacts. Older entries get archived to `agents/<name>/files/memory-archive/`.

**Long reflections from you** (strategic thinking, planning) are stored two ways: a structured summary inside MEMORY, plus the raw verbatim text in `agents/<name>/files/verbatim/<date>.md`. Verbatim files are never touched by compaction.

**Sensitive stuff** (passwords, IDs, secrets) is never written to MEMORY.

### How shared knowledge works (`shared/`)

`shared/` holds anything multiple agents need — facts, rules, your identity — so it stays consistent across the system. Two tiers:

**Always loaded (Tier 0)** — small, every agent sees it on bootstrap:
- `shared/INDEX.md` — catalog of everything in `shared/`
- `shared/user-profile.md` — who you are, how to address you, language preferences
- `shared/conventions.md` — project rules (file naming, dates, markdown style, git)

**Loaded when relevant (Tier 1)** — bigger, only loaded when the topic comes up:
- `shared/truth/team.md` — team roster
- `shared/truth/glossary.md` — acronyms, codenames, internal terms
- `shared/truth/projects.md` — active projects
- `shared/truth/sources/` — verbatim originals (PDFs, transcripts) after main ingests them

**Drop folders for you to use:**
- `shared/import/` — drop a document you want main to process. Then say *"main, ดู `shared/import/<file>` ให้หน่อย"*. main extracts the structured facts into `truth/*.md`, moves the original to `truth/sources/`, deletes the file from `import/`.
- `shared/assets/` — drop binary files (photos, logos, diagrams) manually. Then say *"main, index รูปนี้"*. main appends a row to `shared/assets/INDEX.md`. main never generates or moves binaries on its own — that's your job.

**The simple decision rule:** *"Does another agent need to know this?"* — yes → `shared/`, no → that agent's MEMORY.

### How agents talk to each other

main is the only AI you talk to by default. When main needs a specialist's help, it spawns the specialist via the Task tool — an in-process call/return. No file-based mailboxes between agents.

Every spawn follows the same contract:

- main passes the target's full context (its 4 files), the task, a **DEADLINE** (default 5 minutes), and `EXPECTED_OUTPUT_FILE` if a file is expected.
- The specialist works, then returns a result ending with a status block:
  ```
  STATUS: ok | partial | failed
  ERROR_TYPE: auth_error | source_unreachable | parse_error | policy_block | internal_error | timeout | other
  REASON: <one short line if not ok>
  ```
- main **verifies the output** — opens the expected file, checks it exists and is non-empty. If verification fails, the result is treated as `failed` regardless of what the specialist claimed.
- **Retry policy depends on `ERROR_TYPE`:**
  - Transient (`source_unreachable`, `timeout`, `internal_error`) → retry once
  - Auth / policy / parse errors → escalate to you immediately (retry won't help)
  - `STATUS: partial` → no retry; you decide whether to continue
  - Max 2 spawns of the same prompt regardless

If the specialist gets stuck mid-task and needs main's input, it ends with:
```
MAIN_QUERY: <question>
PARTIAL_RESULT: <what's done so far>
```
main answers, re-spawns the specialist with the answer + partial result, and the work continues. Cap: 3 round-trips per single delegation.

> **Background:** This contract descends from an older file-based "ACP" protocol. The file-based transport was dropped (Task tool replaces it), but the status codes, deadline, error taxonomy, and verification survived — because each one prevents a real failure mode (silent corruption, runaway tasks, blind retries).

### Direct Mode — talking to a specialist directly

Default flow: you ↔ main only. If you want low-latency back-and-forth with a specialist (e.g., a research agent during a deep investigation), open Direct Mode:

- **Open:** `/connect <agent>` (or `main เริ่ม <agent>`)
- **Close:** `/back` — the specialist writes a session summary to its MEMORY, main reads the tail and gives you a recap

While in Direct Mode:
- The specialist talks to you directly. All actions log to that specialist's MEMORY.
- System-level requests (rule changes, spawning new agents) bounce back: *"That requires main — say `/back` first."*

State is tracked in `agents/main/files/.direct-mode-state.json`. If a session crashes mid-Direct-Mode and the state file is older than 24 hours, main resets it on next bootstrap and warns you (no auto-resume).

### Two hats for main

main has two modes:
- **Operating hat** (default) — runs work, delegates, manages day-to-day
- **Design hat** — helps you design / edit the system itself (add agents, change rules, export/import blueprints)

Switch with `main design` (or `/design`) and `main exit design` (or `/operate`). Check current hat with `/status`.

System file edits and creating new agents require Design hat — main will refuse them in Operating hat with a hint to switch.

---

## Hard rules (always-on, override anything)

1. Speak root's preferred language ({{LANGUAGE}}). Match technical depth on demand.
2. Classify every action by Sheridan L1/L2/L3/L4 before executing.
3. Single writer per MEMORY — only the owning agent writes its own MEMORY; sub-agents never write any MEMORY.
4. Never act as root toward externals (email, Slack, calendar invites, social posts). Drafts OK; root sends.
5. Append meaningful actions to own MEMORY before replying. (Meaningful = L2+, decisions, root facts, completed tasks.)
6. Use the Task tool for inter-agent work — no file-based inbox/outbox.
7. Single Gateway — only main talks to root by default; Direct Mode is the opt-in exception.
8. Confirm before mutating system files (README, any AGENT/SKILL/POLICY). Show diff first.
9. Verbatim Reference Pattern — root reflections kept as summary in MEMORY + raw text in `agents/<name>/files/verbatim/`.
10. Single-branch convention — work on the `main` git branch only.
11. Data placement rule — facts other agents need go in `shared/`; per-agent experience goes in that agent's MEMORY.

## Export / Import agents (cross-workspace portability)

Move an agent (or main itself) between Pantheon workspaces as a **blueprint** —
project-specific content (names, dates, MEMORY, URLs) is stripped; only role,
skills, workflows, rules, and personality patterns are kept.

> **Both `/export-agent` and `/import-agent` are Design-hat operations.**
> Switch first with `main design` (or `/design`). main will refuse them in
> Operating hat.

### Filename & path conventions

| | Path | Filename |
|---|---|---|
| **Export output** | `agents/<agent-name>/files/agent-blueprint/` | `<archetype>.v<version>.blueprint.md` |
| **Import input** (drop here) | `shared/imported-agent-blueprint/` | `<archetype>.v<version>.blueprint.md` |

- `<archetype>` = the blueprint's `blueprint_name` (kebab-case role, e.g.
  `research-analyst`, `meeting-notes`).
- `<version>` uses dashes instead of dots — `1.0.0` → `v1-0-0`. So a full
  filename looks like `research-analyst.v1-0-0.blueprint.md`.
- A "handle" is the bare `<archetype>.v<version>` part — used as the
  `/import-agent` argument.

### Export

In Design hat:

```
/export-agent <agent-name>
```

What main does:
1. Verifies Design hat (refuses otherwise).
2. Reads the target agent's 4 files (`AGENT/SKILL/POLICY/MEMORY`).
3. Runs the verbatim export prompt at
   `agents/main/files/export-agent-prompt.md` — main introspects (if exporting
   itself) or spawns a sub-agent loaded with the target's full context.
4. Validates the blueprint (start/end + 4 section delimiters present).
5. Parses meta to derive the filename `<archetype>.v<version>.blueprint.md`.
6. Scans for project-specific leakage and flags anything suspicious.
7. Creates `agents/<agent-name>/files/agent-blueprint/` if missing and saves
   the blueprint there. If a same-name file exists, asks: overwrite or bump version.
8. Reports the saved path and the matching import command for the destination
   workspace.

Sheridan **L2** — read-only on the target, writes only inside that agent's own
`files/agent-blueprint/`.

A blueprint is a single self-contained markdown file delimited by
`===PANTHEON-BLUEPRINT-START===` … `===PANTHEON-BLUEPRINT-END===`, containing a meta
header plus four sections (`AGENT.md`, `SKILL.md`, `POLICY.md`, `MEMORY.md`)
with placeholders (`{{AGENT_NAME}}`, `{{ROOT_NAME}}`, `{{ROOT_ADDRESS}}`,
`{{ROOT_PRONOUN}}`, `{{ROOT_LANGUAGE}}`, `{{TODAY}}`, `{{PROJECT_CONTEXT}}`)
that the new owner must fill at import time.

### Import

In the **destination** workspace (must already be a Pantheon repo):

1. Drop the blueprint file into `shared/imported-agent-blueprint/`. Keep the
   filename intact: `<archetype>.v<version>.blueprint.md`.
2. Switch to Design hat: `main design` (or `/design`).
3. Run:

   ```
   /import-agent <archetype>.v<version>
   ```

   …e.g. `/import-agent research-analyst.v1-0-0`. Run `/import-agent` with no
   argument to have main list every blueprint currently in the drop folder.

What main does (full procedure in `agents/main/files/import-agent-prompt.md`):
1. Verifies Design hat (refuses otherwise).
2. Resolves the handle to
   `shared/imported-agent-blueprint/<handle>.blueprint.md` and validates the
   delimiters.
3. Parses the meta block; defaults the agent name to
   `recommended_default_name`, asks you to choose another if it collides.
4. Pulls placeholder values from `shared/user-profile.md` + today's date and
   asks you for `{{PROJECT_CONTEXT}}` (one line).
5. Splits into the 4 files and substitutes placeholders.
6. **Shows you the diff and asks for confirmation** (Sheridan L3).
7. On confirm, writes `agents/<name>/{AGENT,SKILL,POLICY,MEMORY}.md` +
   `files/.gitkeep`, registers the agent in main's MEMORY roster, and appends
   a README changelog entry.
8. Suggests `/connect <name>` to take the new agent for a test run.

### Rules of thumb

- **Design hat only** — both export and import. Operating hat refuses.
- **A blueprint is a template, not a backup.** MEMORY is intentionally empty —
  that history belongs to the workspace you exported from.
- **Re-export after big changes.** If you reshape an agent's SKILL or POLICY,
  bump `blueprint_version` and re-export so other workspaces can pick up the
  new template.
- **Review before sharing.** Open the saved blueprint end-to-end before sending
  it to someone else — main flags obvious leakage but you are the final reviewer.
- **Filename is load-bearing.** The `<archetype>.v<version>.blueprint.md` shape is
  how the importer locates and identifies blueprints — don't rename ad-hoc.

### Blueprint Lineage System (BLS) — when two workspaces evolve the same agent

The first time you export an agent, main mints a permanent **lineage ID** (a UUID) and starts a **revision history** (chained content hashes). Both travel inside every blueprint. They live on disk at `agents/<name>/files/.lineage.json`.

What this buys you:

- **`/import-agent` is no longer always "create new".** main detects what relationship the incoming blueprint has to your local agent and picks the right mode automatically — CREATE, FAST-FORWARD, MERGE, or REJECT.
- **You can update an agent in-place across workspaces.** Workspace A exports `v1-2-0`, workspace B already has the same agent at an older revision in the same lineage → import fast-forwards.
- **You can merge divergent forks.** Both workspaces edited the agent independently after a common ancestor → main runs a section-level 3-way merge, asks you to resolve only the real conflicts, then writes a new merge revision whose history records both parents.
- **Cross-lineage merges are forbidden.** If two agents share an archetype name but were minted independently (different `lineage_id`), main refuses to merge and offers to import as a new agent under a different name. This prevents accidentally fusing unrelated agents.

#### How main decides what to do at import time

| Local state at this name | Incoming blueprint | What happens |
|---|---|---|
| Doesn't exist | any | **CREATE** — scaffold + seed lineage |
| Legacy (no `.lineage.json`) | legacy (format 1.x) | Ask: rename / overwrite / abort |
| Legacy | modern (format 2.0) | Ask: rename / adopt incoming lineage and overwrite / abort |
| Has lineage, different `lineage_id` | modern | **REJECT** — offer rename-as-new |
| Has lineage, incoming hash already in local history | modern | **NO-OP** — already up to date (or older) |
| Has lineage, local hash is in incoming history (incoming is newer) | modern | **FAST-FORWARD** — overwrite, extend history |
| Has lineage, both diverged from a common ancestor | modern | **MERGE** — interactive 3-way |

#### Merge mechanics

main parses each file (AGENT/SKILL/POLICY — never MEMORY) into sections by markdown heading, classifies each section as no-change / one-side-changed / both-changed, and then walks you through only the real conflicts:

```
CONFLICT — SKILL.md §Skill 5: review_drafts
--- ancestor (revision 1111aaaa) ---
…
--- local (revision 2222bbbb) ---
…
--- incoming (revision 6666cccc) ---
…
Choose: (a) keep local (b) take incoming (c) edit manually (paste new content) (d) abort merge
```

- Sections only one side touched are auto-merged (no question asked).
- Cap: 10 conflicts per merge attempt — if you exceed, main suggests breaking the merge into smaller passes.
- Merge result becomes a new revision whose `revision_history` records both parents — so future workspaces can fast-forward off it.

#### What main reports

After every export / import / merge:

```
Lineage: l-7a3f9c12  (full: 7a3f9c12-4e8b-4a01-9f3c-2d7e1a8b9c4d)
Revision: a3f9c1b2   (history: 1111aaaa → 2222bbbb → a3f9c1b2)
Mode: CREATE | FAST-FORWARD | MERGE | NO-OP
Saved: agents/<name>/
```

You can tell at a glance whether two blueprints belong to the same family — same `Lineage:` short id = mergeable, different = not.

#### Backward compatibility

- Blueprints exported before BLS (no `blueprint_format` / no lineage fields) still import — they go through CREATE mode only and pick up a fresh lineage on import.
- Local agents created before BLS have no `.lineage.json` yet. The first time you export or merge them, main creates one (treating the current state as a new genesis OR offering to adopt an incoming lineage — your choice).
- Re-exporting an agent whose content didn't actually change is **refused** to keep history clean.

> Authoritative spec: [`agents/main/files/blueprint-lineage-spec.md`](agents/main/files/blueprint-lineage-spec.md). main reads it during every export and import.

### Blueprint registry — sharing blueprints across workspaces

The registry is a **folder** that holds blueprints, shared between your workspaces by whatever transport you like (Git, Dropbox, iCloud, USB stick, NFS — anything). main provides convenience commands for `git`-backed registries (push / pull / status); for non-Git transports, the same commands degrade to plain file copy and root keeps the folder synced themselves.

**Setup once per workspace:**

```
main, setup blueprint registry git@github.com:you/pantheon-blueprints.git
```

What main does:
- Asks transport (`git` or `folder`).
- For `git`: clones into `shared/blueprints-registry/` using your system Git auth (SSH keys, `gh auth`, etc. — main never handles credentials).
- Writes `.gitattributes` inside the clone with `*.blueprint.md merge=ours` — Git refuses to auto-merge blueprints. **Conflicts always go through BLS merge in main, never Git's text merger** (which would silently corrupt the structure).
- Writes `shared/blueprints-registry.config.json` (transport, path, branch, layout).
- Adds `shared/blueprints-registry/` to your workspace's `.gitignore` (it is its own Git repo, must not be nested).

**Push a blueprint:**

```
main, push blueprint research
```

main re-runs the privacy/leakage scan (never trusts the scan from export time), copies the file into the registry path, commits with a descriptive message, then asks **a separate confirm** before `git push` — pushing publicly is a deliberate action, not a side effect.

**After every `/export-agent`** main also asks once: *"Push to registry now? (y/n)"* — never auto-pushes. Disable this prompt by setting `auto_suggest_push_after_export: false` in the config.

**Pull updates:**

```
main, pull blueprints
```

main `git fetch`es, shows ahead/behind, asks before `git pull --ff-only`, then scans the registry and classifies every blueprint against your local agents:

| Class | Meaning |
|---|---|
| **NEW** | Registry has an agent you don't have locally — CREATE candidate |
| **UPDATE-FF** | Same lineage, registry revision is newer than local — fast-forward candidate |
| **UPDATE-MERGE** | Same lineage, both diverged — merge candidate (interactive) |
| **NO-OP** | Already at this revision (or older) — nothing to do |
| **CROSS-LINEAGE** | Same archetype name but different `lineage_id` — REJECT (offers rename-as-new) |

You pick which to import. Each import flows through the existing BLS path (Skill 10) — so you get the same CREATE / FAST-FORWARD / MERGE handling whether the blueprint came from a manual drop or from the registry.

**Registry status (read-only):**

```
main, registry status
```

Shows: ahead/behind counts, which registry blueprints are newer than your local agents, which are available to install.

**Refuse rules**

- Push is refused if the local blueprint's content hash doesn't match `.lineage.json.current_revision` (i.e., someone edited the file outside main's export). Re-export properly first.
- Push is refused if the privacy scan flags any leakage.
- Pull `git pull --ff-only` refuses on Git conflicts in the registry — root reconciles in the registry directly (it is their Git repo) before retrying.
- Setup refuses on a non-empty, non-Git path — pick a fresh location.
- main never touches Git credentials, never edits remotes, never runs Git ops outside the registry path.

**Why this design**

- **Transport-agnostic** — the registry is "just a folder". Git is the most common backend; Dropbox / iCloud / USB / NFS all work too.
- **BLS owns semantics, registry owns transport.** `.gitattributes merge=ours` ensures Git stays out of the way at conflict time.
- **Public actions are explicit.** Push to remote requires a second confirm (separate from the local commit confirm). Privacy scan re-runs on every push.
- **No auto-sync.** Bootstrap doesn't pull. Export doesn't auto-push. All registry ops are root-triggered.

> Authoritative spec: [`agents/main/files/blueprint-registry-spec.md`](agents/main/files/blueprint-registry-spec.md). MVP supports one registry per workspace; multi-registry deferred until needed.

## Kernel updates — patching your pantheon when the kernel evolves

Your pantheon was bootstrapped at a specific kernel version (recorded in `.pantheon-kernel-version` at the workspace root). When the kernel author publishes a new version with new rules, new shared/ structure, new policy rows, etc., you can pull those changes in **per-entry, with confirmation** — never wholesale, never automatic.

Two slash commands:

| Command | What it does | Hat |
|---|---|---|
| `/update-status` | Fetch CHANGELOG and show summary of what's pending. **Read-only — does not apply anything.** | Operating OK |
| `/update` | Full interactive walkthrough — apply per entry with confirm. | Design hat |

Conversational equivalents: `main, kernel update status` / `main, check kernel updates`.

What main does:

1. Reads `.pantheon-kernel-version` to know your current version.
2. Fetches the kernel repo's `CHANGELOG.md` (raw URL — single network call, no clone).
3. Parses every version newer than yours into structured **entries**, each tagged with one of:

   | Tag | Meaning |
   |---|---|
   | `[NEW-FILE]` | A new file or folder was added to the kernel |
   | `[NEW-POLICY-ROW]` | One row added to a POLICY table |
   | `[NEW-RULE]` | One new entry in a numbered hard-rule list |
   | `[REPLACE-FILE]` | An existing file is rewritten — needs 3-way merge |
   | `[EDIT-SECTION]` | A specific section inside a file is rewritten — needs 3-way merge |
   | `[REMOVE]` | A file or section is removed |
   | `[MIGRATION]` | Data movement / restructure required (script-form steps) |
   | `[META]` | Documentation note, no action |

4. Shows you a compact summary per version. You pick: review now / defer / cancel.
5. For each entry you review, main proposes the change (preview / diff / 3-way merge), asks **L3 confirm**, and applies it.
6. Versions are applied **oldest-first** (semver ordering — preserves migration dependencies).
7. After applying, `.pantheon-kernel-version` is updated:
   - `kernel_version` bumps to the latest fully-applied version
   - `applied_patches` records which versions were processed
   - `skipped_entries` records what you chose to skip and why (so main doesn't nag you again)

### What main NEVER touches via kernel update

Even if a CHANGELOG entry tries, main refuses and surfaces the refusal:

- Any `MEMORY.md` (yours or any agent's)
- `shared/user-profile.md`
- `shared/truth/*`, `shared/import/*`, `shared/assets/*`
- `agents/<name>/files/.lineage.json`
- Any agent folder that is NOT `main` (your custom agents are off-limits)
- `agents/<name>/files/verbatim/*`

If you genuinely need a kernel patch to touch one of these, apply it yourself.

### When does it run?

- **Only when you ask.** Bootstrap never auto-checks (no surprise network calls, no slowed greeting).
- Re-running is idempotent — applied entries are not re-applied; skipped entries stay skipped unless you reset.

### Aux commands

- `main, kernel version` — print the current `.pantheon-kernel-version` JSON
- `main, kernel skip list` — show entries you've skipped, offer to revisit each
- `main, kernel changelog [<version>]` — fetch and show the CHANGELOG (or one version block) without applying anything

### No revert built-in

If you regret a patch, use Git on your workspace itself (`git revert`, `git checkout`). KUS does not implement revert — your workspace is already a Git repo, that's the right tool.

### If you cloned from a fork

`.pantheon-kernel-version` ships with `kernel_repo` set to the canonical repo URL. If you cloned from a fork, edit that file and point `kernel_repo` at your fork (the raw CHANGELOG URL is derived from there).

> Authoritative spec: [`agents/main/files/kernel-update-spec.md`](agents/main/files/kernel-update-spec.md). CHANGELOG format: see the kernel repo's `CHANGELOG.md` header.

## Re-export the kernel

In Design hat, type `root export pantheon kernel` → main regenerates `installer/artifacts/` + `PANTHEON-INSTALL.md`, stripping personal data, ready for someone else to clone.

## Changelog

| Date | Change |
|---|---|
| {{TODAY}} | Initial scaffold from Pantheon kernel 0.3.1 |
