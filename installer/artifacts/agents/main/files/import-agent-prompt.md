# Import-Agent Procedure

> Used by `main` Skill 10 (`import_agent`). Run this when root provides a
> Pantheon blueprint handle. **Design hat only.**

## Convention

- Blueprint file location (fixed): `shared/imported-agent-blueprint/<handle>.blueprint.md`
- Handle format: `<archetype>.v<version-with-dashes>` — e.g. `research-analyst.v1-0-0`
- Invocation: `/import-agent <handle>` (without `.blueprint.md` suffix and without
  the directory prefix — both are implicit).

If `shared/imported-agent-blueprint/` does not exist, create it before checking
for the file (so the next import has a clear drop point).

## Steps

1. **Verify Design hat.** If Operating, refuse: *"Import creates a new agent —
   switch to Design hat first (`main design` or `/design`)."*

2. **Resolve the blueprint path** from `$ARGUMENTS`:
   - Accept bare handle: `research-analyst.v1-0-0` → resolves to
     `shared/imported-agent-blueprint/research-analyst.v1-0-0.blueprint.md`.
   - Accept handle with `.blueprint.md`: strip and resolve as above.
   - If `$ARGUMENTS` is empty → list every `*.blueprint.md` already present in
     `shared/imported-agent-blueprint/` and ask root which one. If the folder
     is empty, instruct root to drop the blueprint there first.
   - Reject absolute paths or anything outside `shared/imported-agent-blueprint/`
     — keep the convention strict.

3. **Read and validate the blueprint.** Must contain both
   `===PANTHEON-BLUEPRINT-START===` and `===PANTHEON-BLUEPRINT-END===` plus the four
   section delimiters. Reject otherwise.

4. **Parse the meta block.** Extract `blueprint_name`, `blueprint_version`,
   `recommended_default_name`, `source_description`. Show to root in a short
   summary block.

5. **Pick agent name.** Default to `recommended_default_name`. If folder
   `agents/<name>/` already exists, ask root for a different name (or to abort).

6. **Collect placeholder values** from current workspace context:
   - `{{AGENT_NAME}}` ← chosen name
   - `{{ROOT_NAME}}`, `{{ROOT_ADDRESS}}`, `{{ROOT_PRONOUN}}`, `{{ROOT_LANGUAGE}}`
     ← from `shared/user-profile.md`
   - `{{TODAY}}` ← today's ISO date
   - `{{PROJECT_CONTEXT}}` ← ask root for one-line project description (or
     pull from `shared/conventions.md` if present)

7. **Split into 4 files** on the section delimiters
   (`===AGENT.md===`, `===SKILL.md===`, `===POLICY.md===`, `===MEMORY.md===`).

8. **Substitute placeholders** in all 4 file contents.

9. **Show diff to root and request L3 confirmation.** New folder + 4 files +
   any CLAUDE.md / README updates. Do not write yet.

10. **On confirm, write:**
    - `agents/<name>/AGENT.md`
    - `agents/<name>/SKILL.md`
    - `agents/<name>/POLICY.md`
    - `agents/<name>/MEMORY.md`
    - `agents/<name>/files/.gitkeep`

11. **Register the agent:**
    - Add row to main MEMORY §6 Agent roster
    - Add delegation triggers / Quick Reference entry to `CLAUDE.md` if root wants
    - Append README changelog: `imported <blueprint_name>@<blueprint_version> as <name>`

12. **Confirm to root** with first-test suggestion (`/connect <name>`).

## Refuse if
- Invoked outside Design hat
- Blueprint file not found at `shared/imported-agent-blueprint/<handle>.blueprint.md`
- Blueprint delimiters missing or mismatched
- Blueprint contains obvious project-specific leakage (specific person names,
  URLs, dates) — flag to root and ask whether to clean before importing
- `<name>` collides with existing agent and root has not chosen replacement
