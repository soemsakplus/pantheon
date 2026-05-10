# Blueprint Lineage System (BLS) — authoritative spec

> Loaded by main during `/export-agent` and `/import-agent` flows. Defines how agents track ancestry across workspaces so blueprints can be **merged** (not just imported as new) when both sides descend from the same genesis.

## 1. Concepts

- **Lineage** — the "DNA" of an agent. Every agent that descends (via export/import) from a single ancestor shares one immutable `lineage_id`. Different `lineage_id` = different agent; merging across lineages is **forbidden**.
- **Revision** — a snapshot of the agent's content (AGENT + SKILL + POLICY sections, MEMORY excluded). Identified by `revision_hash`.
- **Revision history** — ordered list of all `revision_hash` values from genesis to current. Used to determine ancestry relationships.

## 2. On-disk lineage file

Path: `agents/<name>/files/.lineage.json`

```json
{
  "lineage_id": "7a3f9c12-4e8b-4a01-9f3c-2d7e1a8b9c4d",
  "current_revision": "a3f9c1b2",
  "revision_history": ["1111aaaa", "2222bbbb", "a3f9c1b2"],
  "imported_from_blueprint": "research-analyst.v1-2-0.blueprint.md",
  "last_export_at": "2026-05-10T14:30:00Z",
  "last_import_at": null
}
```

Field rules:
- `lineage_id` — UUID v4, generated **once** at first export of a fresh agent. Immutable thereafter.
- `current_revision` — the hash of the agent's content as it is **right now on disk**.
- `revision_history` — chronological list. Append-only. Never reorder, never remove entries (they are needed for ancestry checks across workspaces).
- `imported_from_blueprint` — filename of the most recent blueprint imported (null if agent was created fresh, never imported).
- `last_export_at` / `last_import_at` — ISO timestamps; null if never happened.

## 3. Hash computation

```
revision_hash = first 8 hex chars of SHA-256( canonicalized_content )
```

**Canonicalization** (apply in order):
1. Concatenate AGENT.md + SKILL.md + POLICY.md content (in that order, with `\n---\n` separator). MEMORY.md is **excluded** from the hash because blueprints intentionally ship empty MEMORY.
2. Strip BOM, normalize line endings to `\n`.
3. Strip trailing whitespace from every line.
4. Strip the `## Version` table at the bottom of each file (version bumps are bookkeeping, not content changes).
5. Strip placeholders' surrounding text only if templated values (e.g. `{{TODAY}}`); placeholders themselves are part of content.
6. Collapse runs of blank lines to a single blank line.

Truncation to 8 hex chars (32 bits) is acceptable: collision risk for a single agent's revision history is negligible; full collision still requires a malicious crafted content. If collision detected at runtime, refuse and report.

## 4. Blueprint meta schema (format 2.0)

```
===PANTHEON-BLUEPRINT-START===
meta:
  blueprint_format: 2.0
  blueprint_name: research-analyst
  blueprint_version: 1.2.0
  exported_at: 2026-05-10T14:30:00Z
  source_description: Research analyst — synthesizes findings from web sources
  recommended_default_name: research
  lineage_id: 7a3f9c12-4e8b-4a01-9f3c-2d7e1a8b9c4d
  revision_hash: a3f9c1b2
  revision_history:
    - 1111aaaa
    - 2222bbbb
    - a3f9c1b2
===AGENT.md===
...
```

Backward compat: blueprints without `blueprint_format`, `lineage_id`, `revision_hash`, or `revision_history` are treated as **format 1.x (legacy)**. Importer falls back to CREATE-only mode (see §6).

## 5. Export orchestration (main side)

1. Verify Design hat (per Skill 9 gate).
2. Run the agent introspection (verbatim prompt in `export-agent-prompt.md`) — receive draft content with placeholders for the 4 sections.
3. Extract `AGENT.md`, `SKILL.md`, `POLICY.md`, `MEMORY.md` sections from the draft.
4. Compute `revision_hash` from canonicalized content (see §3).
5. Read `agents/<name>/files/.lineage.json` if present:
   - **If present and `current_revision == new revision_hash`** → refuse export: *"Content unchanged since last revision (`<hash>`). Bump version is bookkeeping only — re-export rejected to keep revision history clean. If you really need to re-emit, edit the agent's content first."*
   - **If present** → reuse `lineage_id`, append new hash to `revision_history`, update `current_revision`, set `last_export_at`.
   - **If absent** → first-ever export: generate UUID v4 as new `lineage_id`, `revision_history = [revision_hash]`, write `.lineage.json` for the first time.
6. Assemble final blueprint with format-2.0 meta + 4 sections.
7. Save to `agents/<name>/files/agent-blueprint/<archetype>.v<version>.blueprint.md` (rules per existing Skill 9 §11).
8. Persist updated `.lineage.json`.
9. Report saved path + lineage_id (short) + revision_hash to root.

## 6. Import orchestration (main side) — decision tree

```
Read incoming blueprint, parse meta.
├─ blueprint_format absent or < 2.0  →  legacy mode (skip lineage checks; CREATE only as today)
└─ blueprint_format == 2.0  →  proceed with BLS

Look up local agent by `recommended_default_name` (or root-overridden name).
├─ NO local agent at that name
│   └─ CREATE mode
│       1. Show blueprint summary (lineage short, revision, version) to root.
│       2. L3 confirm.
│       3. Write 4 files + write .lineage.json copying lineage_id, current_revision = incoming revision_hash, revision_history = incoming history.
│
└─ Local agent EXISTS
    ├─ Local has NO .lineage.json (legacy local agent)
    │   └─ ASK root (no auto-default, irreversible action):
    │       (a) "Create as new agent under different name"  → branch to CREATE with rename
    │       (b) "Adopt incoming lineage and OVERWRITE local content"  → write 4 files + write .lineage.json from incoming. L3 confirm with extra warning that local edits will be lost.
    │       (c) "Abort"
    │
    └─ Local has .lineage.json
        ├─ local.lineage_id != incoming.lineage_id
        │   └─ REJECT — different lineages cannot be merged.
        │       Offer: "import as new agent under a different name" → branch to CREATE with rename.
        │
        └─ local.lineage_id == incoming.lineage_id  →  same family
            ├─ incoming.revision_hash == local.current_revision
            │   └─ NO-OP — already at this revision. Inform root.
            │
            ├─ incoming.revision_hash ∈ local.revision_history  (incoming is older)
            │   └─ Inform root: "this blueprint is older than local agent; ignoring." NO-OP.
            │
            ├─ local.current_revision ∈ incoming.revision_history  (incoming is newer; fast-forward)
            │   └─ FAST-FORWARD
            │       1. Show diff (local vs incoming).
            │       2. L3 confirm.
            │       3. Overwrite 4 files. Update .lineage.json: current_revision = incoming hash, revision_history = incoming history.
            │
            └─ Neither contains the other  →  diverged with common ancestor
                ├─ Find common ancestor (latest hash present in BOTH histories).
                │   ├─ No common ancestor  →  REJECT (lineage mismatch despite shared id — corruption; surface to root).
                │   └─ Found ancestor `A`.
                │
                └─ MERGE mode (§7)
```

## 7. Merge mode — section-level 3-way

1. Build three trees (ancestor, local, incoming) by parsing AGENT.md / SKILL.md / POLICY.md into sections (top-level `## N. Title` and per-skill `### Skill N: name`).
2. For each section in the union of the three trees, classify:
   - **No change either side** → keep ancestor (no-op).
   - **Only local changed** → take local (auto).
   - **Only incoming changed** → take incoming (auto).
   - **Both changed** → CONFLICT.
   - **Section deleted on one side, edited on other** → CONFLICT.
   - **Section added on only one side** → take that side (auto).
3. For each CONFLICT, present to root conversationally:
   ```
   CONFLICT — <file> §<section title>
   --- ancestor ---
   <text>
   --- local ---
   <text>
   --- incoming ---
   <text>
   Choose: (a) keep local (b) take incoming (c) edit manually (paste new content) (d) abort merge
   ```
4. Cap: **max 10 conflicts per merge.** If more, suggest aborting and merging in smaller doses.
5. After all conflicts resolved, assemble merged content.
6. Compute new `revision_hash` from merged content.
7. Build new `revision_history` = union(local.history, incoming.history) ordered chronologically by appearance, then append the new merge hash. The new revision is a **merge revision** (logically has two parents).
8. Show final summary (sections changed / conflicts resolved / new hash) → L3 confirm.
9. Write 4 files + update `.lineage.json` (`current_revision` = new hash, `revision_history` = merged + new, `last_import_at` = now, `imported_from_blueprint` = incoming filename).

## 8. Refuse rules (summary)

- Cross-lineage merge → REJECT (offer rename-as-new).
- Same lineage, no common ancestor found in histories → REJECT (corruption).
- Re-export with no content change → REJECT (refuse to bloat history).
- Import format 2.0 blueprint into legacy local agent without explicit root choice → REJECT (must choose adopt/rename/abort).
- More than 10 conflicts in one merge → REJECT (suggest splitting).

## 9. Reporting

After every export / import / merge, main reports:
```
Lineage: l-7a3f9c12 (full: 7a3f9c12-...)
Revision: a3f9c1b2  (history: 1111aaaa → 2222bbbb → a3f9c1b2)
Saved: <path>
```

So root can immediately tell whether two blueprints share a lineage just by the short id.
