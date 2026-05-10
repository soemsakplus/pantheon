# INSTALL.md — Pantheon Bootstrap Orchestration

> Script for Claude Code to follow during first-time install.
> Triggered by root `CLAUDE.md`. Files to scaffold live in `installer/artifacts/` — they are **literal templates copied verbatim**, NOT to be regenerated from your head.

## Phase 1 — Greet & ask Q1

**Trigger (exact, case-insensitive):** `pantheon install`

If the user types anything else, do NOT enter install flow — defer to root `CLAUDE.md` "Start ONLY on the explicit install trigger" section (kernel-author mode).

When the trigger fires, in your first response:
1. Greet briefly in **both English and Thai** (1-2 lines each):
   - Identify as "Pantheon installer"
   - Mention: 4 short questions, then automatic system build
2. **Same response**: ask Q1 (language) in both EN and TH per `installer/questions.md`.
3. Wait for user's reply.

After Q1 reply, switch all subsequent conversation to the user's chosen language.

## Phase 2 — Ask Q2-Q4

Ask one at a time per `installer/questions.md`:
- Q2: name + preferred address
- Q3: pronoun
- Q4: role / what you do

Brief acknowledgment after each answer (one short sentence). Hold answers in working memory — do NOT write to disk yet.

## Phase 3 — Confirm

Show summary:

```
- Name: <name>
- Preferred address: <address>
- Pronoun: <pronoun>
- Language: <language>
- Role: <role>
```

Ask user to confirm (in their language: "ยืนยันตามนี้ไหม? / Confirm?").
If they want to edit a field: re-ask that field, then re-confirm.
**Only proceed to Phase 4 on explicit yes.**

## Phase 4 — Scaffold (artifact copy + placeholder injection)

**Important:** All scaffolded files come from `installer/artifacts/`. Do NOT generate file content from your head. Copy verbatim, then replace placeholders.

### 4.1 Copy artifact tree to repo root

Preferred mechanism (Unix shell):

```bash
cp -R installer/artifacts/. .
mkdir -p agents/main/files/memory-archive
touch agents/main/files/memory-archive/.gitkeep
```

This copies:

```
installer/artifacts/CLAUDE.md                 → ./CLAUDE.md           (overwrites installer trigger)
installer/artifacts/README.md                 → ./README.md
installer/artifacts/agents/main/AGENT.md      → ./agents/main/AGENT.md
installer/artifacts/agents/main/SKILL.md      → ./agents/main/SKILL.md
installer/artifacts/agents/main/POLICY.md     → ./agents/main/POLICY.md
installer/artifacts/agents/main/MEMORY.md     → ./agents/main/MEMORY.md
installer/artifacts/shared/user-profile.md    → ./shared/user-profile.md
installer/artifacts/shared/conventions.md     → ./shared/conventions.md
installer/artifacts/.claude/commands/*.md     → ./.claude/commands/*.md
```

Plus creates empty: `agents/main/files/memory-archive/.gitkeep`

**If shell copy is unavailable**, fall back to per-file Read + Write (still verbatim — do not paraphrase).

### 4.2 Replace placeholders

After copying, find-and-replace these tokens in EVERY scaffolded file:

| Token | Value source |
|---|---|
| `{{NAME}}` | user's name from Q2 |
| `{{ADDRESS}}` | user's preferred address from Q2 |
| `{{PRONOUN}}` | user's pronoun from Q3 |
| `{{LANGUAGE}}` | user's language from Q1 |
| `{{ROLE}}` | user's role from Q4 |
| `{{TODAY}}` | today's date in `YYYY-MM-DD` (use environment context — currently 2026 era) |

**Files known to contain placeholders:**
- `CLAUDE.md`
- `README.md`
- `agents/main/AGENT.md`
- `agents/main/SKILL.md`
- `agents/main/POLICY.md`
- `agents/main/MEMORY.md`
- `shared/user-profile.md`

**Mechanism options (pick whichever is reliable in your environment):**
- Unix shell: `find . -type f -name "*.md" -not -path "./installer/*" -not -path "./PANTHEON-INSTALL.md" -exec sed -i '' "s/{{NAME}}/<value>/g" {} \;` (repeat for each token)
- Or per-file Edit tool with `replace_all=true` for each token

### 4.3 Verify

Run a quick scan:

```bash
grep -rE "\{\{[A-Z_]+\}\}" --include="*.md" . | grep -v "installer/" | grep -v "PANTHEON-INSTALL.md"
```

Should return nothing. If any `{{TOKEN}}` remains, fix it before proceeding.

### 4.4 Write `.pantheon-kernel-version` (Kernel Update System bootstrap)

Create `.pantheon-kernel-version` at repo root with this content (substitute `{{TODAY}}` with the actual ISO timestamp at install time):

```json
{
  "kernel_version": "0.2.0",
  "kernel_repo": "https://github.com/soemsakplus/pantheon",
  "installed_at": "{{TODAY}}T00:00:00Z",
  "last_patch_check": null,
  "applied_patches": ["0.2.0"],
  "skipped_entries": []
}
```

This file lets `main` (Skill 13 `manage_kernel_updates`) discover newer kernel versions later via `main, check kernel updates`. Spec: `agents/main/files/kernel-update-spec.md`.

**If the user cloned from a fork** rather than the canonical repo, prompt them: *"Did you clone from `https://github.com/soemsakplus/pantheon`? If you cloned from a fork, edit `.pantheon-kernel-version` and set `kernel_repo` to your fork URL."* — and proceed with the canonical URL as the default.

**Kernel author note:** when releasing a new kernel version, bump the literal `0.2.0` above (and in the version footers in `installer/artifacts/CLAUDE.md` §9, all `agents/main/{AGENT,SKILL,POLICY,MEMORY}.md` §Version tables, and root `README.md` "Current kernel" line). Add the new version block to root `CHANGELOG.md` per its strict format (see CHANGELOG header).

## Phase 5 — Cleanup

1. **Root `CLAUDE.md`** — already replaced in Phase 4.1 (you copied `installer/artifacts/CLAUDE.md` over the installer trigger). Done.
2. **Delete the entire `installer/` directory** — including `artifacts/`, `INSTALL.md`, `questions.md`, `README.md`. Use `rm -rf installer` or equivalent.
3. **Keep `PANTHEON-INSTALL.md`** at repo root as concept reference for future re-export.

After cleanup, repo root should look like:

```
.
├── PANTHEON-INSTALL.md
├── CLAUDE.md                    # post-install version
├── README.md
├── .claude/commands/            # 9 slash commands
├── agents/main/                 # 4 files + files/memory-archive/.gitkeep
└── shared/                      # user-profile.md + conventions.md
```

(no `installer/` anymore)

## Phase 6 — Enter Design hat automatically

Do NOT wait for the user to type `main design`. Instead:

1. Announce in user's language: "ติดตั้งเสร็จแล้ว — เข้า Design hat อัตโนมัติ" (or English equivalent)
2. Behave as `main` in Design hat per the newly-scaffolded `CLAUDE.md` §4.2:
   - Briefly recall PANTHEON-INSTALL.md concept (you've already read it)
   - Address root by their preferred address (`{{ADDRESS}}` value)
   - Show system snapshot:
     - Current agents: just `main`
     - Profile loaded: yes (name, language, role)
     - Pending: no specialist agents yet
   - Offer 2-3 starter suggestions in user's language:
     - "create a notes agent" / "สร้าง agent สำหรับจดโน้ต"
     - "create a research agent" / "สร้าง agent ช่วยทำ research"
     - "tell me more about Pantheon first" / "อธิบาย Pantheon ให้ฟังก่อน"
3. Wait for user input.

## Hard rules during install

- **Never skip Phase 3 confirmation.**
- **Never scaffold partial state.** If interrupted mid-install, restart from Phase 1.
- **Never write user secrets** (passwords, API keys, tokens) anywhere. Refuse and warn.
- **Never invent file content.** All scaffold comes from `installer/artifacts/`.
- **Replace all placeholders before finishing Phase 4.**
- **Single-branch convention** applies from day 1 — do not create branches.
