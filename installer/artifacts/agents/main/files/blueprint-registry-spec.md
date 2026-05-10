# Blueprint Registry — spec (MVP)

> Loaded by main during Skill 12 (`manage_blueprint_registry`) flows. Defines how blueprints sync between workspaces via a shared folder — which may or may not be a Git working tree.

## 1. Concept (transport-agnostic)

A **registry** is just a folder that holds blueprint files. main treats it as a file store; how that folder gets to other machines is **not main's job**:
- Could be a Git clone (most common)
- Could be a Dropbox / iCloud / OneDrive sync folder
- Could be a USB drive
- Could be an NFS / SMB share

main provides convenience operations (push, pull, status) **only** when the registry is configured as a Git repo. For non-Git transports, the same push/pull commands degrade to "copy file" — root keeps the folder synced via whatever tool they like.

**BLS still owns semantics.** This spec only handles file movement. Conflict resolution, lineage, merge — all stay in the existing BLS flow (`blueprint-lineage-spec.md`).

## 2. Folder layout

```
shared/
├── blueprints-registry.config.json     # main reads on every registry op (see §3)
├── blueprints-registry/                # the registry working folder (cloned/synced)
│   ├── .gitattributes                  # if Git transport: refuses auto-merge of .blueprint.md
│   ├── README.md                       # human-readable index (optional)
│   └── <archetype>/                    # folder per archetype
│       ├── <archetype>.v<version>.blueprint.md
│       └── ...
└── imported-agent-blueprint/           # existing — manual drop zone for /import-agent
```

Add `shared/blueprints-registry/` to the workspace's `.gitignore` — it is a separate Git repo (or a Dropbox folder, etc.) and must NOT be nested into the workspace's own Git history.

## 3. Config schema (`shared/blueprints-registry.config.json`)

```json
{
  "transport": "git" | "folder",
  "path": "shared/blueprints-registry",
  "remote_url": "git@github.com:user/pantheon-blueprints.git",
  "branch": "main",
  "layout": "by-archetype",
  "auto_suggest_push_after_export": true
}
```

Field rules:
- `transport`:
  - `"git"` → main may run `git pull` / `git add` / `git commit` / `git push` inside `path`
  - `"folder"` → main only `cp` files into `path`; root syncs the folder by other means
- `path` — relative to workspace root; default `shared/blueprints-registry`
- `remote_url` — only meaningful when `transport: "git"`. main never edits Git remotes.
- `branch` — default `main`
- `layout`:
  - `"by-archetype"` (default) → `<archetype>/<archetype>.v<version>.blueprint.md`
  - `"flat"` → `<archetype>.v<version>.blueprint.md`
- `auto_suggest_push_after_export` — when true, Skill 9 ends with a one-line prompt: *"Saved. Push to registry now? (y/n)"* — never auto-pushes without confirm.

## 4. Setup flow (`main, setup blueprint registry [<git-url>]`)

1. Verify Design hat.
2. If `shared/blueprints-registry.config.json` already exists → ask root: reconfigure or abort.
3. Ask root for transport (`git` or `folder`) if not implied by argument.
4. If `git`:
   - If `<git-url>` provided → `git clone <url> shared/blueprints-registry` (use system git auth — main does not handle credentials).
   - If no url → ask root to either provide one, or set `transport: "folder"` and configure `path` to an existing folder.
   - Write `.gitattributes` inside the clone:
     ```
     *.blueprint.md merge=ours
     ```
     This tells Git **never to auto-merge** blueprint files. Conflicts must go through BLS merge in main, not Git's text merger.
5. If `folder`:
   - Confirm the path exists and is writable.
6. Write `shared/blueprints-registry.config.json` with the chosen settings.
7. Append `shared/blueprints-registry/` to `.gitignore` of the workspace if missing.
8. Report config + suggest first action: *"Try `main, registry status` to see what's available, or `main, push blueprint <name>` to share one."*

**Sheridan:** L3 (creates files in `shared/`, modifies `.gitignore`, may run `git clone`). Show diff + confirm.

## 5. Push flow (`main, push blueprint <agent-name>`)

1. Verify Design hat.
2. Read config; load latest blueprint of `<agent-name>` from `agents/<agent-name>/files/agent-blueprint/` (highest version with current `revision_hash` per `.lineage.json`).
3. **Re-run privacy/leakage scan** — never trust the scan from export time. Scan the file's content for any value matching root profile / `shared/private/` / known PII markers. If anything matches, abort and surface the matches.
4. Compute destination path inside `path` per `layout`:
   - `by-archetype`: `<path>/<archetype>/<archetype>.v<version>.blueprint.md`
   - `flat`: `<path>/<archetype>.v<version>.blueprint.md`
5. If destination exists with identical content → no-op, inform root.
6. Show diff (or "new file") → **L3 confirm**.
7. Copy file into place.
8. If `transport: "git"`:
   - `git -C <path> add <relative>`
   - `git -C <path> commit -m "Push <archetype> v<version> rev <revhash> from <workspace-name>"`
   - Ask root: *"Commit created. Push to remote now? (y/n)"* — only push on yes.
9. If `transport: "folder"`:
   - Done after copy. Remind root to sync the folder via their chosen tool.
10. Report final state + remote ref (if pushed).
11. Append entry to own MEMORY.

**Sheridan:** L3. Push to remote requires a second confirm (separate from the local commit confirm) to make the public action explicit.

## 6. Pull flow (`main, pull blueprints` or `main, pull blueprint <archetype>`)

1. Verify Design hat.
2. Read config.
3. If `transport: "git"`:
   - `git -C <path> fetch`
   - Show summary: ahead/behind counts, new files, modified files. Ask root: *"Apply pull? (y/n)"*
   - On yes: `git -C <path> pull --ff-only` (refuse on conflicts — root must reconcile manually before retrying).
4. If `transport: "folder"`:
   - Skip git ops; assume root already has fresh content in `path`.
5. **Scan registry** — for each `*.blueprint.md` in `<path>`:
   - Parse meta (lineage_id, revision_hash, version)
   - Look up local agent by `recommended_default_name`
   - Classify each registry blueprint:
     - **NEW** — no local agent at this name → CREATE candidate
     - **UPDATE-FF** — same lineage, registry hash newer than local → fast-forward candidate
     - **UPDATE-MERGE** — same lineage, diverged → merge candidate
     - **NO-OP** — already at this revision (or older)
     - **CROSS-LINEAGE** — same name, different lineage → REJECT (rename-as-new offered)
6. Show table of candidates to root. For each, ask: import / skip.
7. For every "import": copy the registry file into `shared/imported-agent-blueprint/<filename>` and trigger Skill 10 (`/import-agent <handle>`) — Skill 10 + BLS handle the actual create / fast-forward / merge.
8. Append summary entry to own MEMORY.

**Sheridan:** L2 to fetch and report. L3 for each individual import (handled by Skill 10).

## 7. Status flow (`main, registry status`)

1. Read config.
2. If `transport: "git"`:
   - `git -C <path> fetch` (silent)
   - Report ahead/behind counts vs `<branch>`.
3. List registry blueprints by archetype, with most recent version.
4. Compare against local agents:
   - For each local agent with a `.lineage.json`: report whether the registry has a newer revision in same lineage.
   - For each registry archetype not present locally: list as "available to install".
5. Suggest next actions.

**Sheridan:** L1 (read-only across the board).

## 8. Refuse rules

- Any registry op without `shared/blueprints-registry.config.json` → tell root to run setup first.
- Push if local blueprint's `revision_hash` ≠ what's in `.lineage.json` (i.e., the file on disk has been edited outside main's export) → REJECT and tell root to re-export properly.
- Push if privacy scan flags any leakage → REJECT, surface matches.
- Pull if `git pull --ff-only` would fail (conflicts in registry) → REJECT, tell root to reconcile in the registry directly (it's their Git repo) before retrying.
- Setup if `path` already exists and is non-empty and non-Git → REJECT, ask root to choose a fresh path.

## 9. Privacy contract

main re-runs leakage scan on **every push** — never trusts a prior scan. Even with `shared/private/` not yet implemented (roadmap), the scan must check root profile fields against blueprint content. If anything matches, the push is aborted with the offending lines surfaced.

main does **not** handle Git credentials or remotes. It assumes the user's environment (SSH keys, `gh auth`, `git credential.helper`) is already set up. If a Git op fails for auth reasons, surface stderr to root and stop.

## 10. Multi-registry (future, NOT in MVP)

This MVP supports **one** registry per workspace. To extend later: replace `path` with an array of `{name, transport, path, remote_url, branch, layout}` entries. Push/pull commands gain a `--registry <name>` argument. Out of scope for MVP — file an issue when first needed.
