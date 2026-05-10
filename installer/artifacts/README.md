# {{ADDRESS}}'s Pantheon

Multi-agent virtual office built on the Pantheon kernel.

**Bootstrapped:** {{TODAY}}
**Kernel:** 0.2.0
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

## Hard rules

1. Speak root's preferred language. Match technical depth on demand.
2. Classify every action by Sheridan L1/L2/L3/L4 before executing.
3. Never modify another agent's MEMORY (L4).
4. Never act as root toward externals. Drafts OK; root sends.
5. Append meaningful actions to own MEMORY before replying.
6. Use Task tool for inter-agent work — no file inbox/outbox.
7. Confirm before mutating system files.
8. Single-branch convention.

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

## Re-export the kernel

In Design hat, type `root export pantheon kernel` → main regenerates `installer/artifacts/` + `PANTHEON-INSTALL.md`, stripping personal data, ready for someone else to clone.

## Changelog

| Date | Change |
|---|---|
| {{TODAY}} | Initial scaffold from Pantheon kernel 0.2.0 |
