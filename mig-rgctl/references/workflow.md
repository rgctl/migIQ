# MigIQ rgctl Workflow

Phase-specific rgctl commands for MigIQ skills. Upstream: [rgctl workflows](https://github.com/rgctl/rgctl/blob/main/skills/rgctl/references/workflows.md) and skills **rgctl-migrate** / **rgctl-review**.

Artifacts live at **`{repo}/.rgctl/`** after `rgctl discover .`. Legacy `~/.rgctl/cache/` is obsolete — re-discover or copy into the repo.

## Artifact Checklist

Run after `discover` (do **not** use `2>/dev/null` on rgctl):

```bash
# 1. Index exists
rgctl -f json metrics --pagerank

# 2. Communities (subsystem mapping)
rgctl -f json communities list | jq '.communities | length'

# 3. Migration plan (when --export-migration-hints was used)
test -f .rgctl/migration_plan.json || test -f .rgctl/dashboard/migration_plan.json

# 4. Plan v2 steps (bounded)
jq '[.steps[] | select(.label != "root")] | .[0:20]' .rgctl/migration_plan.json
```

**Standing rule:** After any MigIQ phase or execute wave that **changes application source files**, run `rgctl update` (optionally `--files …`) **before** the next phase, wave, or graph query. Do not wait until `status` already shows stale — large migrations drift the graph every batch of edits.

If the working tree is dirty and `rgctl -f json status` shows `index_current: false`, run `rgctl update` once before re-querying.

---

## Phase 1: migiq — Full Migration Analysis

**Goal:** Knowledge graph + migration plan + security/taint for downstream skills.

```bash
cd <app-repo-root>

rgctl discover . --with-cfg --with-security --with-taint \
  --with-dashboard --with-harmonic --export-migration-hints \
  --migration-preset hybrid_default --migration-order scheduled
```

**Validate:**

```bash
rgctl -f json metrics --pagerank --communities | jq '.pagerank.top[:5]'
rgctl -f json communities list
rgctl -f json gql --macro-name all_communities unused
jq '[.steps[] | select(.label != "root") | {step, label, schedule_step}] | .[0:20]' \
  .rgctl/migration_plan.json
```

| Flag | Why MigIQ needs it |
|------|-------------------|
| `--with-cfg` | Whole-repo CFG/PDG for taint / mutations / batch inspect |
| `--with-security` | Secrets scanning for containerize/deploy |
| `--with-taint` | Data-flow / injection risks |
| `--with-harmonic` | Migration hotspot ranking |
| `--export-migration-hints` | `.rgctl/migration_plan.json` (v2 `.steps[]`) for mig-plan |
| `--with-dashboard` | Human-readable migration dashboard |

---

## Phase 2: mig-prompt-builder — Current-State Summary

**Prerequisite:** Phase 1 discover completed (or equivalent).

```bash
rgctl -f json metrics --pagerank --communities
rgctl -f json communities list
rgctl -f json gql "MATCH (n:Function) RETURN n LIMIT 20"
rgctl -f json gql "MATCH (a)-[r:USES|DEPENDSON]->(b) RETURN a,b LIMIT 50"

jq '[.steps[] | select(.label != "root")] | .[0:20]' .rgctl/migration_plan.json
```

**Deduce for migration-prompt.md:**

- Languages, frameworks, build system
- Monolith vs modular (community count, pagerank)
- Integration points (USES/DEPENDSON)
- Risk hotspots (pagerank / harmonic)
- Suggested migration **order from `.steps[]`** (not v1 `packages[]`)
- Technical risks (security, taint, Kantra if `--with-kantra`)

---

## Phase 3: mig-plan — Task & Story Generation

**Prerequisite:** `mig-prompt-workspace/migration-prompt.md` + indexed graph.

```bash
rgctl -f json communities list
rgctl -f json gql --macro-name all_communities unused
rgctl -f json blast-radius <Symbol> --depth 2

# Map non-root steps → task groups
jq '[.steps[] | select(.label != "root") | {step, label, schedule_step, priority_rank}]' \
  .rgctl/migration_plan.json
```

---

## mig-containerize — Containerization Readiness

**Prerequisite:** Graph indexed. Prefer discover with `--with-cfg --with-security --with-taint --with-harmonic` if not already done.

```bash
rgctl -f json metrics --pagerank
rgctl -f json communities list    # STOP if empty — re-discover with --with-harmonic
rgctl -f json gql "MATCH (a)-[r:USES|DEPENDSON]->(b) RETURN a,b LIMIT 50"
rgctl -f json semantic query "file path upload storage localhost" --limit 10
```

Map communities → container boundaries. Shared coupling: `communities list` + `blast-radius`.

---

## mig-execute — During Implementation

```bash
# Before changing a symbol
rgctl -f json blast-radius <Symbol> --depth 2

# Call neighborhood
rgctl -f json gql "MATCH (a:Function)-[:CALLS*1..3]->(b:Function) WHERE a.name = '<fn>' RETURN a,b LIMIT 50"

# One-off data flow / slice — lazy CFG OK (optional --function / --with-alias)
rgctl -f json slice <file> --line <n> --variable <var>
rgctl -f json cpg flows <file> --line <n> --variable <var> --direction forward
```

Reserve rediscover `--with-cfg` for taint/mutations/batch inspect — not for every slice.

### After each migration wave / phase that edits sources (required)

```bash
rgctl update   # required — keep structural graph current for the next wave/phase

# Example: javax → jakarta
rgctl -f json find "import javax*" --type import   # expect total == 0

# Optional call-path deltas (upstream rgctl-review)
rgctl -f json review paths --synthetic-head worktree --max-symbols 25
```

Same rule after mig-test-gen / mig-containerize / mig-deploy when they write into the app tree (not workspace-only markdown).

---

## mig-test-gen / mig-deploy

```bash
rgctl -f json gql "MATCH (f:Function) WHERE f.community_id = '<id>' RETURN f LIMIT 20"
rgctl -f json gql "MATCH (a)-[r:USES|DEPENDSON]->(b) RETURN a,b LIMIT 50"
rgctl -f json metrics --pagerank
```

---

## Discover Flag Cheat Sheet (by MigIQ stage)

| Stage | Recommended discover flags |
|-------|---------------------------|
| migiq Phase 1 | `--with-cfg --with-security --with-taint --with-dashboard --with-harmonic --export-migration-hints` |
| mig-prompt-builder | *(reuse Phase 1 index)*; plan via `.steps[]` |
| mig-plan | *(reuse)* + `.rgctl/migration_plan.json` v2 |
| mig-containerize | `--with-cfg --with-security --with-taint --with-harmonic` (if re-indexing) |
| mig-execute | `rgctl update` after edits; lazy `slice`/`cpg flows`; `--with-cfg` for taint/mutations |
| Java framework migration | add `--with-kantra --kantra-target <target>` |

Full flag reference: [rgctl Migration Feature-Flag Cheat Sheet](https://github.com/rgctl/rgctl/blob/main/skills/rgctl/references/workflows.md#migration-feature-flag-cheat-sheet).
