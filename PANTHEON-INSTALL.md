# PANTHEON — Multi-Agent Virtual Office Kernel

> **Concept and architecture reference.**
> For installation, see `/installer`. For day-to-day usage after install, see `CLAUDE.md`.

**Version:** kernel-0.3.0
**License:** MIT (suggested — adjust for your use)
**Lineage:** evolved from `dvm-workspace` v0.22 (extracted 2026-05-09 as kernel-0.1.0, refactored 2026-05-10 as kernel-0.2.0, KUS shipped 2026-05-10 as kernel-0.3.0).

---

## 1. What is Pantheon

A **multi-agent virtual office** for Claude Code. You (root — the human) work with **main** (your chief of staff), who orchestrates **multi-agents** (specialists with memory) who in turn spawn **sub-agents** (ephemeral workers).

The system is **file-first** — every agent is a directory of plain markdown. State persists in git. Claude Code's `CLAUDE.md` bootstrap is the entry point.

---

## 2. Roles

> **Canonical vocabulary (solid — used everywhere in code, triggers, agent files, conversation):**
> `root` · `main` · `multi-agents` · `sub-agents`
>
> `root` = the human user — think "superuser, but as a human." The Unix `root` account has supreme authority over a machine; in Pantheon, `root` is the supreme authority over the multi-agent system, namely you.
>
> Any analogies in §3 (Jesus/apostles, founder/contractors, …) are explanatory aids only. They are NEVER substituted into code, prompts, file names, or triggers.

| Role | Who | Persona | Memory |
|---|---|---|---|
| **root** | the human user (also "me" / "owner" / "operator" in casual speech) | unchanging — never an AI persona | external (your brain) |
| **main** | chief of staff agent | one persona, two hats (Operating / Design) | persistent (`MEMORY.md`) |
| **multi-agents** | specialists you grow over time | each their own | persistent |
| **sub-agents** | ephemeral workers spawned via Task tool | task-scoped | none |

---

## 3. Main's two hats

`main` is one persona that switches between two **roles** depending on what root is doing:

- **Operating hat** (default): execute work, delegate to multi-agents, manage day-to-day
- **Design hat**: help root design/edit the system itself — create new agents, edit rules, refine policy

`main` **never pretends to be root**. Root is always you, the human at the keyboard. "Design Mode" means main wearing the Design hat — not main impersonating root.

**Mental models (optional analogies — explanation only):** the 4-tier structure mirrors patterns from human institutions. ⚠️ Reminder: canonical terms stay `root` / `main` / `multi-agents` / `sub-agents`. The columns below are *intuition pumps*, not alternative names — never use them in code, prompts, or triggers.

| Canonical (use this) | Religious (Christian) — intuition | Corporate — intuition | Unix / OS — intuition |
|---|---|---|---|
| **root** | God | founder / owner | `root` superuser (as a human) |
| **main** | Jesus (sole intermediary, serves God) | chief of staff / executive assistant | shell / `init` / `pid 1` |
| **multi-agents** | apostles / disciples (named, persistent) | department heads / named specialists | system services / daemons |
| **sub-agents** | day laborers / hired hands (one task, no continuity) | contractors / freelancers | child processes (`fork()` → exit) |

All four columns describe the **same structural pattern**: one supreme principal → one gateway intermediary → a persistent team of specialists → many ephemeral workers below. The Unix column especially aligns with Pantheon's OS-installer metaphor — and the canonical term `root` is borrowed directly from there.

---

## 4. 4-File architecture per agent

Every agent has exactly four `.md` files:

- **`AGENT.md`** — *who am I?* (identity, role, personality)
- **`SKILL.md`** — *what can I do?* (tools, skills, SOP)
- **`POLICY.md`** — *should I / am I allowed?* (Sheridan L1-L4 permissions)
- **`MEMORY.md`** — *what did I do?* (activity log + key decisions + facts)

Plus an optional `files/` directory for journals, archives, generated artifacts.

---

## 5. Communication

| Path | Mechanism |
|---|---|
| root ↔ main | direct conversation (Claude Code CLI) |
| main → multi-agent | **Task tool** — main reads target's full 4-files, embeds in Task prompt, receives result via Task return |
| main → sub-agent | **Task tool** — ephemeral worker, returns result directly |
| multi-agent → sub-agent | same — Task tool, returns to spawner |
| root ↔ multi-agent | **Direct Mode** (opt-in) — temporary direct chat |

**Deprecated:** file-based `inbox/` / `outbox/` folders (kernel-0.1.x). Removed in 0.2.0 because Claude Code is single-process — Task tool returns results directly. For future async / cron-triggered messaging, the extension point is `agents/<name>/files/messages/` (not scaffolded by default).

---

## 6. Triggers

| Command | Effect |
|---|---|
| `main start` / `/main` / `/mn` / `เริ่ม` | activate main, Operating hat |
| `main design` / `/design` / `เข้า design mode` | switch main to Design hat |
| `main exit design` / `/operate` | switch main back to Operating hat |
| `main เริ่ม <agent>` / `/connect <agent>` | enter Direct Mode with a multi-agent |
| `กลับ main` / `/back` | exit Direct Mode |
| `main status` / `/status` | report current hat / mode |

**Note:** `root` is implicit (root is you, always). Commands address `main`, not "root". Aliases like `root start` / `root add agent` may be kept for backward compat but the canonical form uses `main`.

---

## 7. Sheridan L1-L4 autonomy framework

Every action classified before execution:

| L | Name | Behavior |
|---|---|---|
| **L1** | Inform | do, then tell |
| **L2** | Suggest | propose plan → confirm → do |
| **L3** | Confirm | state action + impact → explicit confirmation |
| **L4** | Block | refuse, root does it themselves |

See each agent's `POLICY.md` for the full permission matrix.

---

## 8. Memory tiering

- **Multi-agents** have persistent `MEMORY.md` — compacted weekly/monthly with rolling summary, archived to `files/memory-archive/YYYY-MM.md`
- **Sub-agents** are stateless — task → output → done. No memory.

---

## 9. Model tiering

- `claude-sonnet-4-x` for judgment / decisions (default)
- `claude-opus-4-x` for deep planning / critical decisions (fallback)
- `claude-haiku-4-x` for cheap parsing / routine sub-agent work

Each agent's SKILL.md / spawn template specifies the model.

---

## 10. Verbatim Reference Pattern

When root types reflections / strategic thoughts, agents keep BOTH a structured summary AND the raw verbatim block (in a triple-backtick code fence) so root can trace every original word. Applies only to root-typed content.

---

## 11. Hard rules (always-on, override anything)

1. Speak root's preferred language by default; technical-deep on demand
2. Classify every action by L1/L2/L3/L4
3. Never modify another agent's MEMORY (L4)
4. Never act as root toward externals (drafts OK, sending is root's job)
5. Append meaningful actions to own MEMORY before replying
6. Use Task tool for inter-agent work — no file inbox/outbox
7. Confirm before mutating system files (any AGENT/SKILL/POLICY of any agent)
8. Single-branch convention — work on `main` git branch only

---

## 12. Installation

The repo ships with a self-contained installer:

```
PANTHEON-INSTALL.md            # this concept doc (kept after install)
CLAUDE.md                       # installer trigger (replaced after install)
installer/
├── INSTALL.md                  # orchestration script (~200 lines)
├── questions.md                # 4 minimum profile questions
├── README.md                   # tester docs
└── artifacts/                  # exact files to copy → injected with placeholders
    ├── CLAUDE.md
    ├── README.md
    ├── agents/main/{AGENT,SKILL,POLICY,MEMORY}.md
    ├── shared/{user-profile,conventions}.md
    └── .claude/commands/*.md
```

Flow: clone → open Claude Code → first message triggers install → 4 questions → confirm → copy artifacts + inject placeholders (`{{NAME}}`, `{{ADDRESS}}`, `{{PRONOUN}}`, `{{LANGUAGE}}`, `{{ROLE}}`, `{{TODAY}}`) → delete `installer/` → enter Design hat automatically.

Design rationale: artifacts are **literal files copied verbatim**, not LLM-generated content. This makes install **deterministic and weak-model-safe** — the LLM's job is orchestration + string replacement, not generation.

---

## 13. Post-install directory structure

```
.
├── PANTHEON-INSTALL.md          # this concept doc (kept for re-export)
├── README.md                    # project log + changelog
├── CLAUDE.md                    # bootstrap (auto-loaded by Claude Code)
├── .claude/
│   └── commands/                # slash commands
│       ├── main.md
│       ├── design.md
│       ├── operate.md
│       ├── back.md
│       ├── status.md
│       ├── connect.md
│       ├── commitpush.md
│       ├── refreshmain.md
│       └── compact.md
├── agents/
│   └── main/
│       ├── AGENT.md
│       ├── SKILL.md
│       ├── POLICY.md
│       ├── MEMORY.md
│       └── files/
│           └── memory-archive/.gitkeep
└── shared/
    ├── user-profile.md
    └── conventions.md
```

When you spawn new agents, each gets its own `agents/<name>/` folder with the same 4 files + `files/`. `shared/` is read by all agents.

---

## 14. Re-export workflow

When you've evolved your kernel and want to share / re-install elsewhere:

1. In Design hat: `root export pantheon kernel`
2. main will:
   - Read all current `agents/*` and `shared/*` files
   - Strip project-specific content (root profile, agents' MEMORY content, your custom triggers)
   - Regenerate `installer/artifacts/` with up-to-date templates (placeholders restored)
   - Regenerate this `PANTHEON-INSTALL.md` with bumped version
3. Review diff → confirm → commit
4. Anyone who clones the repo can run the installer to bootstrap their own pantheon

---

## 15. What's NOT in this kernel (intentionally — for you to build)

- **Specialist agents** (no daily, news, journal, etc.)
- **Orchestrated workflows** (sync/async parallel agent fan-out)
- **Compaction algorithm details** (full algo — see SKILL.md)
- **External MCP integrations** (Gmail, Calendar, Slack — wire per agent)
- **Cross-machine agent communication** (future protocol)
- **Agent personality drift / "growth" over time**
- **Domain-specific Hard Rules** (e.g., your industry's compliance — add to CLAUDE.md §7)
- **Cron / scheduled agents** (planned for kernel 0.3.x)
- **Agent blueprint import/export** (planned for kernel 0.3.x)

These are deliberately omitted to keep the kernel **lean and portable**.

---

## 16. Background reading

- Sheridan 1992 — *Telerobotics, Automation, and Human Supervisory Control*. MIT Press.
- Parasuraman, Sheridan, Wickens 2000 — "A model for types and levels of human interaction with automation," *IEEE Trans Sys Man Cyb 30(3)*.
- Anthropic 2024 — *Constitutional AI*.
- Park et al. 2023 — "Generative Agents: Interactive Simulacra of Human Behavior," arXiv:2304.03442.
- Packer et al. 2023 — "MemGPT: Towards LLMs as Operating Systems," arXiv:2310.08560.
- Vogels 2009 — "Eventually Consistent," *Communications of the ACM 52(1)*.

---

## 17. Pantheon kernel version log

| v | Date | Note |
|---|---|---|
| 0.1.0 | 2026-05-09 | Initial kernel — extracted from `dvm-workspace` v0.22. Single-file install spec with embedded fenced blocks. |
| 0.1.1 | 2026-05-10 | Mode trigger semantic fix: swap `main start` (Operating) ↔ `root start` (Design). |
| 0.2.0 | 2026-05-10 | **Major refactor.** Concept: root is always the human, never an AI persona; main has two hats (Operating/Design). Communication: deprecated file inbox/outbox; all inter-agent work via Task tool. Install: split into orchestration (`installer/INSTALL.md`) + literal artifacts (`installer/artifacts/`) for deterministic, weak-model-safe install. New slash commands: `/main`, `/design`, `/operate`, `/connect`. PANTHEON-INSTALL.md role changed: from scaffolding spec to concept reference. |
| 0.3.0 | 2026-05-10 | **KUS (Kernel Update System) ships.** Added: Skill 13 `manage_kernel_updates` + `/update-status` + `/update` slash commands, root `CHANGELOG.md` with strict tag format, `kernel-update-spec.md`, `.pantheon-kernel-version` workspace state file. Also bundled (post-0.2.0 work documented as part of this release): M1-M13 protocol tightening, shared knowledge layer (Tier 0/1, `truth/`, `import/`, `assets/`), Blueprint Lineage System (BLS), blueprint registry MVP. First version with patchable upgrades — future releases (0.4.0+) reachable via `/update` from any 0.3.0+ install. |
