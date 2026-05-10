# MEMORY.md — `main`

Experience layer. AGENT=identity, SKILL=tools, POLICY=permissions.
**Compaction:** weekly summary after 7d, monthly after 30d, archive at `files/memory-archive/`.

## 1. Recent activity log
> Newest → oldest. Format: `- [YYYY-MM-DD] action → outcome`

- [{{TODAY}}] Bootstrap — installed via Pantheon installer (kernel 0.2.0). Profile collected from {{ADDRESS}}.

## 2. Key decisions
> *Never compact this section.*

### Architecture
- **4-file architecture** — AGENT (who) / SKILL (what) / POLICY (should/may) / MEMORY (did).
- **Two-hat main** — Operating hat (default work) / Design hat (system design). root is the human, never an AI persona.
- **Memory tiering** — multi-agent has long-term memory; sub-agent stateless.
- **Single Gateway with Direct Mode exception** — root ↔ main by default; opt-in Direct Mode for direct chat with multi-agents.
- **Inter-agent comm via Task tool** — no file inbox/outbox. Spawner reads target's full context, gives task via Task prompt, receives result via Task return.

### Models
- main: `claude-sonnet-4-x` primary, `claude-opus-4-x` fallback for deep planning
- Sub-agents: Haiku for routine (logging, parsing), Sonnet for judgment

### Decision framework
- **Sheridan L1-L4** (1992) applied to every action

### Operational rules
- **Verbatim Reference Pattern** — root-typed reflections kept structured + raw verbatim block
- **Single-branch convention** — no feature branches

## 3. Learned facts about root
> *Never compact.* Source of truth: `shared/user-profile.md`.

- **Name:** {{NAME}}
- **Preferred address:** {{ADDRESS}}
- **Pronoun:** {{PRONOUN}}
- **Default language:** {{LANGUAGE}}
- **Role:** {{ROLE}}

## 4. Open items
> Pending decisions / awaiting root input.

- [ ] root to design first specialist agent (Design hat ready — say `main design`)

## 5. Active projects
> What root is working on.

- (fill as root grows)

## 6. Agent roster

| Name | Type | Status | Folder | Notes |
|---|---|---|---|---|
| `main` | Chief of Staff | Active | `agents/main/` | self |

## 7. Rolling summaries

### Weekly
*(none yet)*

### Monthly
*(none yet)*

## 8. Archive pointer
*(no archives yet)*

## 9. Version

| v | Date | Note |
|---|---|---|
| 0.2.0 | {{TODAY}} | Bootstrapped from Pantheon kernel 0.2.0 |
