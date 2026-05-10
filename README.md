# Pantheon — Multi-Agent Virtual Office Kernel

> **An OS-like scaffolding for Claude Code.**
> Install once, then grow your own pantheon of AI agents — one specialist at a time.

This repo is **the installer**, not a running system. Cloning it gives you the install media; running the installer gives you a fresh, almost-empty pantheon to grow into your own.

---

## What is Pantheon?

In Greek and Roman mythology, the **Pantheon** was the temple housing all the gods. In this project, **you** are root — and Pantheon is the temple you build, agent by agent.

Pantheon is a kernel that bootstraps a multi-agent system inside Claude Code. After installation, you get a single agent (`main`) — your chief of staff — ready to listen, take orders, delegate, and (in Design hat) help you create more specialist agents over time.

---

## Philosophy: the OS metaphor

Think of this repo as an **OS installer ISO**:

| Computing | Pantheon |
|---|---|
| Burning the OS installer | Cloning this repo |
| Running setup → empty filesystem + shell | Running the installer → empty `agents/` + `main` |
| Installing apps over time | Adding multi-agents, skills, MCP connectors over time |
| Setup wizard disappears after install | `installer/` deletes itself; only the running OS remains |
| The shell (bash, zsh) | `main` — minimal but complete, ready to extend |

After install you have an **empty virtual office**. You staff it agent by agent. Over time it becomes a personal team that knows your work, your tools, your style.

---

## The cast

> **Canonical terms — the only ones used in code, triggers, file names, and conversation:**
> **`root`** · **`main`** · **`multi-agents`** · **`sub-agents`**
>
> `root` = the human user (think "superuser, but as a human"). The Unix `root` account has supreme authority over a machine; in Pantheon, `root` is the supreme authority over the multi-agent system — and that's **you**, sitting at the keyboard.
>
> These four words are the **working vocabulary** of the entire system. Every AGENT.md, SKILL.md, POLICY.md, slash command, and trigger uses these exact terms. The analogies further down (Jesus/disciples, founder/contractors, etc.) are **explanatory aids only** — never substitute them in code, prompts, or instructions.

| Role | Who | Persona | Memory |
|---|---|---|---|
| **root** | **you** (the human user) — also "me" / "owner" / "operator" in casual speech | unchanging — never an AI persona | external (your brain) |
| **main** | chief of staff agent | one persona, two hats | persistent (`MEMORY.md`) |
| **multi-agents** | specialists you grow (notes, research, accountant, …) | each their own | persistent |
| **sub-agents** | ephemeral workers spawned via Task tool | task-scoped | none |

### Main's two hats

- **Operating hat** (default): execute work, delegate, manage day-to-day
- **Design hat**: help root design/edit the system itself — create new agents, edit rules, refine policy

**Critical:** `main` is **never** root. Root is always you, the human at the keyboard. "Design Mode" means `main` wearing the Design hat to help you, *not* `main` impersonating you.

### Mental models (optional analogies — for explanation only)

> ⚠️ **Reminder:** the canonical terms remain **`root` / `main` / `multi-agents` / `sub-agents`**. The two columns below are *teaching aids* — useful when introducing the system to someone new. They are **not** alternative names. Do not use these words in code, prompts, agent files, or triggers.

If the 4-tier structure feels abstract, two analogies may help:

| Canonical (use this) | Religious (Christian) — for intuition | Corporate / professional — for intuition | Unix / OS — for intuition |
|---|---|---|---|
| **root** | God (supreme being) | founder / owner | `root` superuser (as a human) |
| **main** | Jesus (sole intermediary, serves God, speaks to disciples on God's behalf) | chief of staff / executive assistant | shell / `init` / `pid 1` |
| **multi-agents** | apostles / disciples (named, persistent, carry forward the mission) | department heads / named specialists | system services / daemons |
| **sub-agents** | day laborers / hired hands (paid for one task, no continuity) | contractors / freelancers | child processes (`fork()` → exit) |

All three right-hand columns describe the **same structural pattern** as the canonical column: one supreme principal → one gateway intermediary → a persistent team of specialists → many ephemeral workers underneath. The Unix column especially fits Pantheon's OS-installer metaphor — and is where the canonical `root` term comes from.

See [PANTHEON-INSTALL.md](PANTHEON-INSTALL.md) for the full architecture spec.

---

## What's in this repo

```
.
├── README.md                     # ← you are here (about the project)
├── PANTHEON-INSTALL.md           # concept + architecture reference (kept after install)
├── CLAUDE.md                     # installer trigger (auto-loaded by Claude Code)
└── installer/
    ├── INSTALL.md                # orchestration script (~200 lines)
    ├── questions.md              # 4 minimum profile questions
    ├── README.md                 # tester / contributor docs
    └── artifacts/                # literal templates — copied verbatim during install
        ├── CLAUDE.md
        ├── README.md
        ├── agents/main/{AGENT,SKILL,POLICY,MEMORY}.md
        ├── shared/{user-profile,conventions}.md
        └── .claude/commands/*.md
```

Nothing here is a running pantheon. Running pantheons live in **other** repos bootstrapped from this one.

---

## How to install

### 1. Copy or clone

```bash
# option A — git clone
git clone <this-repo-url> my-pantheon
cd my-pantheon

# option B — local copy (no git history)
cp -R /path/to/pantheon /path/to/my-pantheon
cd /path/to/my-pantheon
find . -name .DS_Store -delete   # macOS noise
```

### 2. Open Claude Code

```bash
claude
```

Claude Code auto-loads [CLAUDE.md](CLAUDE.md), which tells the assistant it's running the Pantheon installer.

### 3. Say hi

Type **anything** — `hi`, `start`, `.`, anything. Claude Code only responds after a user message, so you need to send one to kick things off. The installer ignores the **content** of your first message and treats it purely as a trigger.

### 4. Answer 4 questions

The installer greets you in **English + Thai**, then asks:

1. Preferred language for the system to talk to you
2. Your name + how you want to be addressed
3. Pronoun
4. Your role / what you do

These are the **minimum** to bootstrap. More fields fill in over time via `shared/user-profile.md`.

### 5. Confirm and watch

After confirmation, the installer:

1. **Copies** `installer/artifacts/` to repo root (verbatim — no LLM rewriting)
2. **Replaces** placeholders (`{{NAME}}`, `{{ADDRESS}}`, `{{PRONOUN}}`, `{{LANGUAGE}}`, `{{ROLE}}`, `{{TODAY}}`) with your answers
3. **Deletes** the `installer/` folder
4. **Replaces** the bootstrap `CLAUDE.md` with the post-install one
5. **Enters Design hat automatically** and introduces itself as `main`

Total time: about 1-2 minutes.

### 6. Start growing

Once in Design hat, ask main to create your first specialist:

```
create a notes agent that helps me journal daily
```

`main` proposes `AGENT.md` / `SKILL.md` / `POLICY.md` / `MEMORY.md` drafts → you review → confirm → done. The agent now lives at `agents/notes/` and can be invoked anytime.

---

## How the install works (under the hood)

The kernel deliberately separates **orchestration** from **content**:

- [installer/INSTALL.md](installer/INSTALL.md) — small, prescriptive script. Tells the LLM the steps.
- [installer/artifacts/](installer/artifacts) — actual template files. **Copied verbatim**, then placeholders are string-replaced.

**Why?** Earlier kernel versions had Claude *generate* the files from a single fenced spec. Weaker models drifted, summarized, or dropped fences. The artifacts approach makes install **deterministic and weak-model-safe** — the LLM does `cp` + sed, not creative writing.

---

## Re-export the kernel

After evolving your own pantheon, you can re-export an updated kernel for others:

```
root export pantheon kernel
```

`main` regenerates [installer/artifacts/](installer/artifacts) and [PANTHEON-INSTALL.md](PANTHEON-INSTALL.md), stripping personal data and bumping the version. Commit, push, share.

---

## Contributing

This repo is the kernel — improvements here propagate to every new pantheon. Good places to contribute:

- **Better questions** in [installer/questions.md](installer/questions.md) (current set is intentionally minimal — adding more should be justified)
- **More slash commands** in [installer/artifacts/.claude/commands/](installer/artifacts/.claude/commands)
- **Refinements** to the four `main` templates: AGENT, SKILL, POLICY, MEMORY
- **Documentation** in [PANTHEON-INSTALL.md](PANTHEON-INSTALL.md) and this README
- **Roadmap features** (see [PANTHEON-INSTALL.md §15](PANTHEON-INSTALL.md)):
  - Cron / scheduled agents
  - Agent blueprint import/export
  - MCP connector declarations
  - Reusable playbooks (cross-agent shared procedures)

### Test flow before submitting

```bash
cp -R . /tmp/pantheon-test
cd /tmp/pantheon-test
find . -name .DS_Store -delete
claude
```

Type `hi`, run through all 4 questions in both Thai and English (try Thai once, English once), confirm install completes, verify `installer/` is gone, verify Design hat self-introduction works.

### Style

- Markdown-first. Plain text wins over fancy formatting.
- Keep `INSTALL.md` orchestration small. Push content to `artifacts/`.
- Preserve placeholder convention: `{{TOKEN}}` in CAPS_WITH_UNDERSCORES.
- Bump kernel version in [PANTHEON-INSTALL.md §17](PANTHEON-INSTALL.md) for any change to the architecture, mode triggers, hard rules, or install flow.

---

## Version

**Current kernel:** 0.2.0 (2026-05-10)

See [PANTHEON-INSTALL.md §17](PANTHEON-INSTALL.md) for the full version log.

---

## References

The system draws on:

- Sheridan 1992 — *Telerobotics, Automation, and Human Supervisory Control* (the L1-L4 autonomy framework)
- Park et al. 2023 — *Generative Agents: Interactive Simulacra of Human Behavior* (memory-based persona persistence)
- Packer et al. 2023 — *MemGPT: Towards LLMs as Operating Systems* (memory tiering)
- Anthropic 2024 — *Constitutional AI* (the rule-based behavior model)

---

## License

MIT (suggested — adjust for your use).
