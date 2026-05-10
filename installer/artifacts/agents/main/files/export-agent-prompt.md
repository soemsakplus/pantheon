# Export-Agent Prompt (verbatim)

> Used by `main` Skill 9 (`export_agent`). When the export skill runs, send the
> prompt below **verbatim** to the agent being exported (after loading its full
> 4-file context: AGENT.md, SKILL.md, POLICY.md, MEMORY.md). The agent
> introspects against this prompt and returns the blueprint.

---

You are about to be EXPORTED as a reusable agent blueprint for a system called Pantheon (multi-agent virtual office, file-first, 4-file architecture per agent).

# Goal
Output ONE markdown blueprint that describes you as a **reusable template** — strip everything project-specific, keep only your transferable skills, workflows, rules, and personality patterns.

# Critical rules

1. **Blueprint, not snapshot** — produce a template that someone could deploy on a fresh machine for an unrelated project. Not a memoir.

2. **Strip ALL of these:**
   - Specific project names, company names, codenames, internal acronyms
   - Specific people's names, emails, handles
   - Past conversations, decisions, log entries, MEMORY content
   - File paths from current workspace
   - URLs, repo names, ticket IDs, channel names
   - Dates of past events
   - Any "we did X last week" / "the user prefers Y because of Z incident" content

3. **Keep ONLY:**
   - Your role / archetype (e.g. "research analyst", "note-taker", "code reviewer") — generic
   - Your skills / capabilities (what you can DO) — described abstractly
   - Your workflows / SOPs (HOW you work) — reusable steps
   - Your tone / personality rules — pattern-level
   - Your decision bias / heuristics
   - Hard rules / things you refuse to do
   - Tools you typically need (described by category, not specific account)

4. **Use placeholders** for anything the new owner must fill in:
   - `{{AGENT_NAME}}` — name in new workspace
   - `{{ROOT_NAME}}` — new owner's name
   - `{{ROOT_ADDRESS}}` — how new owner wants to be called
   - `{{ROOT_PRONOUN}}`
   - `{{ROOT_LANGUAGE}}`
   - `{{TODAY}}` — install date
   - `{{PROJECT_CONTEXT}}` — brief project description (filled at install)

5. **MEMORY stays empty** — only schema/structure, no entries.

# Output format — ONE single markdown file with these 4 sections separated EXACTLY by these delimiters:


```
===PANTHEON-BLUEPRINT-START===
meta:
  blueprint_name: <short kebab-case archetype, e.g. "research-analyst", "meeting-notes">
  blueprint_version: 1.0.0
  exported_at: <ISO date>
  source_description: <one line — what kind of agent this is, generic>
  recommended_default_name: <suggested name in kebab-case, e.g. "research">
===AGENT.md===
# AGENT.md — `{{AGENT_NAME}}`

## 1. Identity
- Name: `{{AGENT_NAME}}`
- Type: <role archetype>
- Reports To: root ({{ROOT_ADDRESS}})

## 2. Role
<one-paragraph generic role description — no project specifics>

## 3. Responsibilities
<bulleted list, transferable>

## 4. Personality
### Tone
<rules, language-agnostic where possible — reference {{ROOT_LANGUAGE}}>
### Style rules
<address as {{ROOT_ADDRESS}}, pronoun {{ROOT_PRONOUN}}, etc.>
### Decision bias
<heuristics>

## 5. Constraints (POLICY.md is source of truth)
<authorized / forbidden — generic>

===SKILL.md===
# SKILL.md — `{{AGENT_NAME}}`

## 1. Capabilities
<what you can do — abstract>

## 2. Tools
<tool categories needed, NOT specific accounts/URLs>

## 3. Workflows / SOPs
<reusable step-by-step procedures>

## 4. Output formats
<templates you produce>

===POLICY.md===
# POLICY.md — `{{AGENT_NAME}}`

## 1. Sheridan levels
- L1 (autonomous): <list>
- L2 (notify): <list>
- L3 (confirm first): <list>
- L4 (forbidden): <list>

## 2. Hard rules
<refuse-to-do, override anything>

## 3. Escalation
<when to hand back to main / root>

===MEMORY.md===
# MEMORY.md — `{{AGENT_NAME}}`

> Empty — populate on first run in new workspace.

## Activity log
_(none yet)_

## Key decisions
_(none yet)_

## Facts about root
_(populate from {{PROJECT_CONTEXT}} on first session)_

===PANTHEON-BLUEPRINT-END===
```


# Final check before output
Re-read your blueprint. If you find ANY of these, remove and replace with placeholder or generic phrasing:
- A specific company / product / project name
- A specific person's name (other than placeholders)
- A specific past event, log entry, or decision
- A specific URL, file path, or account
- Any sentence that begins with "We" / "Last time" / "The user once" / "In project X"

Then output the blueprint and NOTHING ELSE. No preamble, no explanation, no closing remarks.
