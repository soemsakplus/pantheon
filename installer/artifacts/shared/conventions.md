# Project conventions

> Cross-cutting rules and conventions all agents read.

## File naming

- **Agent journals / reports:** `agents/<name>/files/{YYYYMM}/{YYYYMMDD}-{slug}.md`
- **Memory archive:** `agents/<name>/files/memory-archive/{YYYY-MM}.md`
- **Imported legacy notes:** `agents/<name>/files/{YYYYMM}/{YYYYMMDD}-imported.md`

## Date format

- **In filenames:** `YYYYMMDD` (ISO 8601 basic, no separators)
- **In content:** `YYYY-MM-DD` (ISO 8601 extended)
- **Time:** `HH:MM` 24-hour, with timezone label (e.g., `14:30 ICT`)

## Markdown style

- **Headers:** `#` for top, `##` for major sections
- **Todos:** `- [ ]` open, `- [x]` done
- **Status icons:** 🔴 urgent | 🟡 follow-up | 🟢 ok | 📝 not started | ✅ done | ⚠️ caution | ⭐ important

## Versioning

- **Files have a `## Version` table at bottom:** `| v | Date | Note |`
- **Bump on:** semantic version (major.minor.patch)
  - **patch:** typo / clarification
  - **minor:** content change, no breaking
  - **major:** breaking change to structure

## Git

- **Single-branch convention:** work on `main` git branch only
- **Commit messages:** present tense, scope-prefixed (`agent: <change>`, `system: <change>`, `docs: <change>`)
- **No force-push to main**

## Verbatim reference pattern

When agents record root-typed reflections / strategic thoughts, they MUST preserve BOTH:

1. **A structured summary** in the agent's MEMORY Activity Log entry.
2. **The raw text** saved to `agents/<name>/files/verbatim/<YYYY-MM-DD>-<slug>.md`.

The MEMORY entry contains a pointer to the verbatim file (e.g., `→ verbatim: files/verbatim/2026-05-10-product-strategy.md`). Verbatim files are never compacted. Applies only to root-typed content, not generated summaries. (See CLAUDE.md hard rule §9.)

Example MEMORY entry:

```markdown
- [2026-05-10] root reflection on Q3 product strategy → key points: pivot to enterprise, freeze consumer features, hire 2 AEs. → verbatim: files/verbatim/2026-05-10-q3-strategy.md
```

## Inter-agent communication

- Use **Task tool** to spawn agents (multi-agent delegation, sub-agent ephemeral work)
- **No file-based inbox/outbox folders** (deprecated since kernel 0.2.0)
- Spawner reads target's full context (AGENT/SKILL/POLICY/MEMORY for multi-agent), embeds in Task prompt, receives result via Task return
- Each agent appends its own actions to its own MEMORY (per its POLICY)
- For future async/cron-triggered communication: use `agents/<name>/files/messages/` (extension point — not scaffolded by default)
