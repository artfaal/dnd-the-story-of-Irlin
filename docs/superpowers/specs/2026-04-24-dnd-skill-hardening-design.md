# dnd Skill Hardening — Design Spec

**Date:** 2026-04-24
**Status:** Approved
**Scope:** Extend the `dnd` skill so an LLM agent operating the wiki cannot silently introduce drift, dangling references, or stale derived data. Wiki data at `${DND_DIR:-$HOME/PROJECTS/dnd-wiki}` is never edited by a human — only by an agent through this skill. The skill must therefore make the contract obvious, fail fast on violation, and provide a one-command fix.

## Motivation

A recent audit of the wiki surfaced issues that the existing linter (9 checks, strict-clean) could not catch:

- `STATE.md` + `CAMPAIGN.md` referenced character slugs (`ashen`, `irroniya-hezer-leonberger`, `dimitrina-speysskaya`) that did not match the actual character files (`ashan`, `ironiya-leonberger`, `dimitrina-speiskaya`). These sat in markdown tables that the linter does not scan.
- `STATE.md session_next.focus_quests` contained `find-ilkai`, a quest that had been `выполнен` since session #3.
- `rumours.md` had a broken YAML fence: half the entries lived outside the ```yaml block, and one `related:` entry referenced `torvin` (not a valid slug).
- `npcs/berser-trant.md` had `relations:` shaped as a list of dicts; the linter's `_iter_refs` only handled dict shape, so a broken reference to `bratstvo-vzirauschih-ochei` slipped through.
- `SCHEMA.md` taxonomy contained near-duplicate tags: `проклятье/проклятие`, `ирлинский-лес/ирлиндский-лес`, `храм/храм-ильматера`, etc.

These issues were fixed by hand in commit `87f737f`. This spec describes the skill-level changes that prevent them from recurring.

## Non-Goals

- Rewrite of existing lint checks (9 checks stay as-is; `_iter_refs` gets one targeted fix).
- Migration of existing rules/homebrew pipeline.
- CI/GitHub Actions setup (local pre-commit hook is enough for v1).
- Performance optimization (lint already < 100ms on current wiki).

## File Contract

Three categories, each with a distinct agent contract:

| Category | Files | Agent may edit? | Validation |
|----------|-------|-----------------|-----------|
| **Source of truth** | `characters/*.md`, `quests/*.md`, `npcs/*.md`, `locations/*.md`, `sessions/*.md`, `rules/*.md`, `items/*.md` | ✅ directly | existing 9 lint checks + `relations-list-shape` fix |
| **Derived** | `index.md`; `STATE.md` derived blocks; `CAMPAIGN.md` derived block | ❌ only via scripts | `state-drift`, `campaign-drift` (fixable via `--fix`) |
| **Manual** | `rumours.md`, `loot.md`, `timeline.md`, `log.md`, manual blocks of `STATE.md`/`CAMPAIGN.md` | ✅ directly | `state-refs`, `rumour-refs`, structural checks |

### Derived-block markers

Derived regions in markdown body are wrapped in HTML comments, matching the convention already used by `index.md`:

```markdown
<!-- BEGIN generated:party — do not edit; run `dnd-sync-state.py` -->
| Персонаж | Уровень | HP max | Статус |
|----------|---------|--------|--------|
| torvin-kamneklyatv | 4 | 38 | проклятье |
...
<!-- END generated:party -->
```

YAML frontmatter cannot carry HTML comments, so derived frontmatter keys are managed differently: `dnd-sync-state.py` rewrites a **fixed set of keys** (`session_next.focus_quests`) while leaving all other keys untouched. The set is hard-coded in the script, documented in `SKILL.md`.

### Split per file

**`STATE.md` derived:**
- Party table (body) — regenerated from `characters/*.md` (slug, level, status → mapping to cursed/healthy/etc.)
- `session_next.focus_quests` (frontmatter) — filtered against `quests/*.md` status; `выполнен`/`провален`/`заморожен` removed

**`STATE.md` manual:**
- `date_ingame`, `party_location`, `current_focus`, `updated`, `session_next.date`
- Sections: "Активные эффекты / угрозы", "Главный квест", "Не забыть в следующей сессии", "Очередь квестов следующей сессии"

**`CAMPAIGN.md` derived:**
- Player/character table (body) — regenerated from `characters/*.md` (player, name, slug, race, class)

**`CAMPAIGN.md` manual:**
- Frontmatter, "Лор-константы", "Особенности"

## Components

### New: `scripts/dnd-sync-state.py`

Idempotent script that regenerates derived blocks in `STATE.md` and `CAMPAIGN.md`.

**CLI:**
```
dnd-sync-state.py            # regenerate (no-op if clean)
dnd-sync-state.py --check    # exit 0 if clean, 1 if drift detected (no writes)
dnd-sync-state.py --fix      # explicit regenerate (same as bare)
dnd-sync-state.py --migrate  # one-shot: insert markers into existing wikis
```

**Behavior:**
- Loads all `characters/*.md` → builds party table
- Loads all `quests/*.md` → status index
- Loads current `STATE.md`/`CAMPAIGN.md`
- For each derived block: compare expected vs actual, write if different
- Logs to `log.md`: `## [ts] sync-state | state=updated|clean | removed_focus=[...]`
- Exit codes: 0 (clean / updated), 1 (check mode + drift), 2 (error — missing markers, schema violation)

**Edge cases:**
- Missing markers → error, suggest `--migrate`
- Empty `characters/` → writes empty table with "(no characters)" placeholder
- Quest status unknown (not in enum) → treats as active, leaves in focus_quests, warns

### New: `scripts/dnd-install-hooks.sh`

Installs `.git/hooks/pre-commit` in `${DND_DIR}`.

**CLI:**
```
dnd-install-hooks.sh          # non-destructive, bails if hook exists
dnd-install-hooks.sh --force  # overwrite existing
```

**Hook contents:**
```bash
#!/usr/bin/env bash
set -e
SKILL_DIR="${DND_SKILL_DIR:-$HOME/.openclaw/skills/dnd}"
python3 "${SKILL_DIR}/scripts/dnd-sync-state.py" --check
python3 "${SKILL_DIR}/scripts/dnd-lint.py" --strict
```

The hook is self-contained. `DND_SKILL_DIR` env overrides the default location. If the skill path is missing, the hook exits 1 with a clear message.

### Extended: `scripts/dnd-lint.py`

**New checks:**

| Name | Severity | Fixable | Detects |
|------|----------|---------|---------|
| `state-drift` | error | ✅ (`--fix` delegates to sync-state) | derived blocks in `STATE.md` don't match source of truth |
| `campaign-drift` | error | ✅ | same for `CAMPAIGN.md` |
| `state-refs` | error | ❌ | `party_location`, `current_focus` point to non-existent slugs |
| `stale-focus` | warning | ✅ | focus_quests contain completed/failed/frozen quests |
| `taxonomy-hygiene` | info | ❌ | near-dups, prefix-containment, cross-category in SCHEMA |
| `rumour-refs` | error | ❌ | `related:` slugs in `rumours.md` YAML blocks don't exist |
| `relations-list-shape` | n/a | n/a | Fix to `_iter_refs` — handles `relations:` as list-of-dicts |

**Did-you-mean** — on any broken-ref / state-refs error, run `difflib.get_close_matches(missing, all_slugs, n=1, cutoff=0.7)` and include suggestion in message.

**`relations-list-shape` fix** — `_iter_refs` extended:

```python
if isinstance(rels, list):
    for item in rels:
        if isinstance(item, dict):
            for rfld, rval in item.items():
                if rfld in RELATION_SINGLE_FIELDS and isinstance(rval, str) and rval:
                    yield f"relations.{rfld}", rval
```

Kept backward-compatible with dict shape.

### Extended: `scripts/dnd-init.py`

New campaigns get:
- `STATE.md` template with `<!-- BEGIN/END generated:party -->` markers
- `CAMPAIGN.md` template with `<!-- BEGIN/END generated:players -->` markers
- `git init` (already done) + `dnd-install-hooks.sh` invocation

### Extended: `SKILL.md`

New section **"Файловый контракт"** — the three-category table from this spec.

New trigger phrases added to command table:

| Phrase | Script |
|--------|--------|
| «синхронизируй state / sync» | `dnd-sync-state.py` |
| «установи хуки / install hooks» | `dnd-install-hooks.sh` |

Compile ritual step 6 updated to insert `dnd-sync-state.py --fix` before `dnd-rebuild.py`.

Explicit prohibition added: **"Agent must not use `git commit --no-verify`."**

## Data Flow

### Compile ritual (updated)

```
1-4. (unchanged) — read, parse, route, write sessions/YYYY-MM-DD.md
5.   UPDATE STATE.md manual zone (effects, reminders, current_focus)
     UPDATE timeline.md, rumours.md, loot.md (manual files)
6.   RUN scripts/dnd-sync-state.py --fix    ← NEW
     RUN scripts/dnd-rebuild.py              (index.md)
     RUN scripts/dnd-lint.py --strict        (blocking)
     (optional) scripts/dnd-embed.py
7-9. (unchanged) — log, commit, report
```

If lint fails at step 6, compile halts; agent fixes and retries.

### Live-edit flow (non-compile)

```
Agent edits quests/find-ilkai.md → status: выполнен
→ git add && git commit
→ pre-commit hook fires:
  1. dnd-sync-state.py --check → drift detected (STATE has find-ilkai as focus) → exit 1
  2. hook prints remediation: "run `dnd-sync-state.py --fix`"
  3. commit blocked
→ Agent runs fix → STATE.md updated → retry commit → green
```

### New campaign flow

```
dnd-init.py --path ~/PROJECTS/new-campaign
→ scaffolds directories
→ writes SCHEMA.md, CAMPAIGN.md, STATE.md (with markers), index.md stub
→ git init
→ dnd-install-hooks.sh
→ ready for agent use
```

## Error Handling

**Lint messages** — one-line error + concrete fix command:

```
ERROR state-drift [STATE.md]
  party table block stale
  fix: python3 scripts/dnd-sync-state.py --fix

ERROR state-refs [STATE.md:party_location]
  slug 'nothern-keller' — page does not exist
  did you mean: northern-keller?

WARNING stale-focus [STATE.md:session_next.focus_quests]
  'find-ilkai' — quest status=выполнен (autofixable)
  fix: python3 scripts/dnd-sync-state.py --fix

INFO taxonomy-hygiene [SCHEMA.md]
  near-dup in status-markers: проклятье ↔ проклятие (levenshtein=1)
  prefix-containment in cities: храм ⊂ храм-ильматера

ERROR rumour-refs [rumours.md:entry#7]
  related: torvin — no such slug
  did you mean: torvin-kamneklyatv?
```

**Pre-commit hook** collects all errors, prints as one block, exits 1. `git commit --no-verify` remains as emergency pedal but is explicitly forbidden for the agent in `SKILL.md`.

**Sync-state error modes:**
- Missing `STATE.md` → exit 2, "STATE.md missing; run `dnd-init.py`"
- Missing markers → exit 2, "migration required; run `dnd-sync-state.py --migrate`"
- Empty `characters/` → exit 0, warning logged
- Unknown quest status → exit 0, warning logged, slug kept in focus_quests

## Testing

Test infrastructure bootstrapped from scratch; the current deployed skill at `~/.openclaw/skills/dnd/` has no `tests/`. The old backup at `~/.openclaw.pre-reinstall-20260414-150346/workspace/skills/dnd/tests/` provides a reference pattern but its code is not reused verbatim.

**Structure:**
```
~/.openclaw/skills/dnd/
├─ pytest.ini
└─ tests/
   ├─ __init__.py
   ├─ conftest.py
   ├─ fixtures/
   │  ├─ minimal-wiki/        # 2 characters, 2 quests, 1 session, SCHEMA, STATE
   │  └─ drift-wiki/          # minimal + known drift cases
   ├─ test_kb_common.py       # frontmatter parser, slugify
   ├─ test_dnd_lint.py        # 9 existing + 7 new checks
   ├─ test_dnd_sync_state.py  # happy, drift, idempotence, missing-markers
   ├─ test_dnd_rebuild.py     # smoke
   ├─ test_dnd_init.py        # bootstrap correctness
   └─ test_install_hooks.py   # idempotence, --force
```

**Pinning existing behavior:**
Before extending `dnd-lint.py`, write golden tests on current behavior against the `minimal-wiki` fixture. This locks the 9 existing checks so we catch any regression from the `_iter_refs` change.

**TDD loop per new check:**
1. Write failing test (fixture triggers violation) → red
2. Implement check → green
3. Refactor if warranted

**Integration test:**
Full compile-like sequence on `drift-wiki`: apply edit → run sync-state → run rebuild → run lint → assert clean + expected diff.

**Coverage rule (not a percentage):**
- Every new check has ≥1 positive (violation) and ≥1 negative (clean) test.
- `dnd-sync-state.py` has an idempotence test (sync → sync = no-op).
- At least one happy-path end-to-end integration test.
- Run time target: < 5 seconds full suite.

## Migration Strategy

1. **Snapshot** current skill: `cp -r ~/.openclaw/skills/dnd ~/.openclaw.pre-hardening-2026-04-24`. Rollback = restore from snapshot.
2. **Add tests directory** — set up pytest, write golden tests for existing 9 checks, confirm green against current `~/PROJECTS/dnd-wiki/`.
3. **Implement `relations-list-shape` fix** first (smallest, highest-risk-to-existing-behavior change), verify golden tests still pass.
4. **Implement new lint checks** one by one, TDD. Each lands with tests.
5. **Implement `dnd-sync-state.py`**, TDD.
6. **Migrate live wiki**: run `dnd-sync-state.py --migrate` on `~/PROJECTS/dnd-wiki/` → inserts markers into current STATE.md/CAMPAIGN.md. Verify `--check` is clean.
7. **Implement `dnd-install-hooks.sh`**, install into live wiki.
8. **Update `dnd-init.py`** bootstrap.
9. **Update `SKILL.md`** with file contract section and new triggers.
10. **Smoke test**: simulate the original audit bugs on a copy of the wiki, confirm lint catches each and `--fix` resolves them.

## Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| `relations-list-shape` fix breaks existing behavior | Golden tests on current wiki before + after |
| Sync-state corrupts STATE.md on bug | `--check` used in pre-commit; `git` provides rollback |
| Agent bypasses hook with `--no-verify` | Explicit prohibition in SKILL.md; behavior baked into prompts |
| Markers accidentally removed | `state-drift` error on next lint; `--migrate` re-inserts |
| New lint overhead slows compile | Checks are O(pages); current load 75 pages < 100ms total |

## Open Questions (resolved)

- **Derived vs validation?** → Derived (user choice).
- **Source repo location?** → Work directly in `~/.openclaw/skills/dnd/`, snapshot before changes.
- **Update `dnd-init.py` this pass?** → Yes, new campaigns get hooks + markers from day 0.
- **Pre-commit hook scope?** → Per-wiki-repo (the wiki is its own git repo; skill is not).
