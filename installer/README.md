# /installer — Pantheon Bootstrap

This folder is the first-time installer for Pantheon. It is **deleted automatically** at the end of installation (Phase 5 in `INSTALL.md`).

## Files

| File / Dir | Role |
|---|---|
| `INSTALL.md` | Orchestration script Claude follows step-by-step |
| `questions.md` | The 4 minimum profile questions |
| `artifacts/` | **Literal templates** copied verbatim to repo root during install |
| `README.md` | This file |

## Design rationale: artifacts vs. generation

Earlier kernel versions had Claude **generate** scaffolded files from a single fenced spec. That was fragile: weaker models drifted, summarized, or dropped fences.

Kernel 0.3.0 splits install into:

1. **Orchestration** (`INSTALL.md`) — small, prescriptive, mostly control flow
2. **Artifacts** (`artifacts/`) — exact files to **copy verbatim**, with `{{PLACEHOLDER}}` tokens replaced by collected user data

This makes install **deterministic and weak-model-safe**: the LLM's job becomes copy + string-replace, not creative writing.

## How install is triggered

The repo root contains a `CLAUDE.md` that Claude Code auto-loads on session start. That file tells Claude to read `installer/INSTALL.md` and follow it.

## Testing the installer

To test in a clean repo:

1. Copy this entire `pantheon/` directory to a new location (e.g. `/tmp/pantheon-test`).
2. Confirm only these exist at root: `PANTHEON-INSTALL.md`, `CLAUDE.md`, `installer/`. (No `agents/`, `shared/`, `.claude/` should be present yet.)
3. `cd` into the new location and open Claude Code.
4. Type exactly `pantheon install` to start the installer — that is the only trigger. Claude responds with the bilingual EN+TH greeting and asks Q1. Any other first message keeps Claude in kernel-author mode (regular assistant on the installer codebase), so contributors can edit installer files without triggering a fresh install. Worth testing both paths.
5. Answer the 4 questions. Confirm the summary.
6. Watch Claude `cp` artifacts → inject placeholders → delete `installer/` → enter Design hat and self-introduce.

## Post-install state

After successful install:

```
.
├── CLAUDE.md                    # post-install bootstrap (from artifacts/)
├── PANTHEON-INSTALL.md          # kept as kernel reference
├── README.md                    # project log
├── .claude/commands/            # 9 slash commands
├── agents/main/                 # 4 files + files/memory-archive/
└── shared/                      # user-profile.md + conventions.md
```

`installer/` is gone.

## Placeholder tokens

These appear in `artifacts/*.md` and are filled at install time:

| Token | Filled from |
|---|---|
| `{{NAME}}` | Q2 (full name) |
| `{{ADDRESS}}` | Q2 (preferred address — what main calls root) |
| `{{PRONOUN}}` | Q3 |
| `{{LANGUAGE}}` | Q1 |
| `{{ROLE}}` | Q4 |
| `{{TODAY}}` | install date in `YYYY-MM-DD` |

After install, the `grep` check in Phase 4.3 ensures no `{{TOKEN}}` remains.
