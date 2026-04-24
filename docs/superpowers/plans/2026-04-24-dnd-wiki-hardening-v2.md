# D&D Wiki Hardening v2 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Harden the `dnd` skill so the LLM agent cannot introduce drift, miss cross-refs, invent undocumented fields, or silently lose uncertain information. Close the 26 data defects and 9 validation gaps surfaced by the audit.

**Architecture:** Six-layer system (contract → data → validation → derivation → tooling → process). Schema expansion first, then unified relations, then data fixes, then open-questions infra, then 9 new lint checks, then expanded sync-state, then items migration, then new tooling, then SKILL.md/README updates, finally full smoke.

**Tech Stack:** Python 3.11+, PyYAML, pytest (existing 52 tests stay green), networkx (new dep for graph), subprocess-based script invocation from tests, markdown with HTML-comment derived blocks.

## Repository layout (critical context)

**Two separate git repos involved:**

- **Data repo:** `/Users/claw/PROJECTS/dnd-wiki/` — SCHEMA.md, all entity files, STATE/CAMPAIGN/timeline/rumours/loot/log, questions.md (new), docs/.
- **Skill repo:** `/Users/claw/.openclaw/` — contains `skills/dnd/{scripts,lib,tests,SKILL.md,README,pytest.ini}`. Note: skill files are currently **untracked** in that repo — we stage them explicitly. Never `git add -A` in .openclaw.

**Commit convention:**
- Data changes → commit from `/Users/claw/PROJECTS/dnd-wiki/`.
- Skill code changes → `cd ~/.openclaw && git add skills/dnd/<specific-files> && git commit`.
- Every commit message uses prefix: `schema:`, `fix:`, `feat:`, `feat(lint):`, `docs:`, `chore:`, `test:`.

**Testing:**
- Tests live in `~/.openclaw/skills/dnd/tests/`. Run from skill root: `cd ~/.openclaw/skills/dnd && python -m pytest`.
- Fixtures in `conftest.py` provide `wiki` pytest fixture (temporary wiki dir with MINIMAL_SCHEMA).
- Subprocess pattern: tests invoke scripts as `subprocess.run([sys.executable, LINT, ...], env={"DND_DIR": wiki})`.

**Pre-commit hook:** installed via `scripts/dnd-install-hooks.sh` in the DATA repo. Runs `dnd-lint.py` (non-strict after Phase 6 — see spec §E). Currently installed and running.

## File structure (all additions/modifications)

### Data repo (`/Users/claw/PROJECTS/dnd-wiki/`)

**Modified:**
- `SCHEMA.md` — expanded with optional fields, new enums, unified relations, reorganized taxonomy
- `STATE.md` — new derived blocks `generated:active_effects`, `generated:confirmed_rumours`; extended `generated:party` with Name column
- `CAMPAIGN.md` — unchanged structure, maybe content updates if migration catches any
- `rumours.md` — unchanged format (already YAML)
- `loot.md` — trimmed after items migration
- `npcs/*.md` (33 files) — migrate relations format, add mentions, fix undeclared fields
- `quests/*.md` (22 files) — migrate relations, fix givers/locations, add next_session where needed
- `locations/*.md` (7 files) — migrate related_npcs → relations format
- `sessions/*.md` (5 files) — mentions backfill, ensure required fields
- `characters/*.md` (5 files) — add hp_max, subclass_status, active_quest_ref where applicable
- `characters/_party.md` — **deleted**

**Created:**
- `questions.md` — wiki-wide open questions with YAML entries
- `items/ring-mari-klodet.md`, `items/bandura-miala.md`, `items/elemental-stone-fire.md`, `items/elemental-stone-air.md`, `items/musical-instrument-1984.md`, `items/hat-of-vredit.md`, `items/mechanical-canary.md`, `items/gold-pipe.md`
- `.index/inbound.json` — reverse-index (gitignored or committed — decide during Phase 6 Task 13)

### Skill repo (`/Users/claw/.openclaw/skills/dnd/`)

**Modified:**
- `scripts/dnd-lint.py` — 9 new checks registered; severity triage in main()
- `scripts/dnd-sync-state.py` — 2 new derived-block generators (active_effects, confirmed_rumours); party table extended
- `scripts/dnd-rebuild.py` — emit `.index/inbound.json`; maybe add Name column to index
- `scripts/dnd-search.py` — faceted filters (`--type`, `--relation`, `--status`, `--location`, `--min-score`, `--explain`, `--include-rumours`)
- `scripts/dnd-embed.py` — new namespaces (rumours, timeline, questions)
- `scripts/dnd-note.sh` — optional `--dedupe` flag
- `scripts/dnd-install-hooks.sh` — update hook to not use `--strict`
- `lib/kb_common.py` — helpers for new parsing (relations unified iter, placeholder detection, inbound index loader)
- `SKILL.md` — 7 new sections per spec C.9
- `README.md` — update CLI reference

**Created:**
- `scripts/dnd-questions.py` — open-questions CLI
- `scripts/dnd-graph.py` — relations graph export
- `scripts/dnd-verify-compile.py` — raw/live vs session diff
- `tests/test_dnd_lint_bidirectional.py`
- `tests/test_dnd_lint_body_mentions.py`
- `tests/test_dnd_lint_session_mentions.py`
- `tests/test_dnd_lint_rumour_reflected.py`
- `tests/test_dnd_lint_timeline_parity.py`
- `tests/test_dnd_lint_item_from_loot.py`
- `tests/test_dnd_lint_placeholders.py`
- `tests/test_dnd_lint_stale_questions.py`
- `tests/test_dnd_lint_undeclared_fields.py`
- `tests/test_dnd_questions.py`
- `tests/test_dnd_graph.py`
- `tests/test_dnd_verify_compile.py`
- `tests/test_dnd_sync_state_expanded.py`
- `tests/test_dnd_search_filters.py`

**Updated fixtures:**
- `tests/conftest.py` — MINIMAL_SCHEMA extended to include new enums and unified relations

---

## Phase 1: SCHEMA expansion

Phase goal: document every field already in use + introduce new enums. No data migration yet; lint continues to pass because new rules are not active until Phase 6.

### Task 1: Extend SCHEMA.md with optional-fields blocks

**Files:**
- Modify: `/Users/claw/PROJECTS/dnd-wiki/SCHEMA.md`

- [ ] **Step 1: Open SCHEMA.md, locate "## Required fields per type" section, insert new "## Optional fields per type" section immediately after it**

Insert this block before the "## Relations policy (YAML-only)" section:

```markdown
## Optional fields per type

Fields listed here are validated by the linter when present (enum/regex applied) but not required. Values must obey the declared type.

<!-- schema:optional:common -->
- aliases
- portrait
- map
- open_questions
<!-- /schema:optional:common -->

<!-- schema:optional:npc -->
- first_seen
- last_seen
- aliases
- portrait
- secrets
<!-- /schema:optional:npc -->

<!-- schema:optional:quest -->
- next_session
- deadline
- blocker
- related_npcs
- related_locations
- created_session
- created_date
- location
<!-- /schema:optional:quest -->

<!-- schema:optional:location -->
- related_npcs
- visited
- map
- parent_region
- sub_locations
<!-- /schema:optional:location -->

<!-- schema:optional:session -->
- players
- absent
- game_day
- location_ingame
- mentions
- tags
<!-- /schema:optional:session -->

<!-- schema:optional:character -->
- hp_max
- subclass
- subclass_status
- age
- portrait
- active_quest_ref
<!-- /schema:optional:character -->

<!-- schema:optional:item -->
- holder
- type_item
- attuned_by
- notes
<!-- /schema:optional:item -->

<!-- schema:optional:rule -->
- source_book
- page
<!-- /schema:optional:rule -->
```

Note: `secrets` is listed as optional for npc ONLY as a migration bridge — it gets removed in Task 11 (Phase 3). After Task 11, remove it from this block.

- [ ] **Step 2: Add new enum blocks immediately after the existing enum section, before the taxonomy block**

Insert after `<!-- /schema:enum:rumour:status -->`:

```markdown
### item (extended)

<!-- schema:enum:item:type_item -->
- оружие
- броня
- артефакт
- расходник
- записка
- ключ
- книга
- инструмент
- документ
- реликвия
- зелье
<!-- /schema:enum:item:type_item -->

### character (extended)

<!-- schema:enum:character:subclass_status -->
- chosen
- tbd
<!-- /schema:enum:character:subclass_status -->

### open_question

<!-- schema:enum:open_question:category -->
- clarify
- lore
- blocker
<!-- /schema:enum:open_question:category -->

<!-- schema:enum:open_question:status -->
- open
- answered
- obsolete
<!-- /schema:enum:open_question:status -->

### relation

<!-- schema:enum:relation:kind -->
- союзник
- враг
- нейтральный
- коллега
- муж
- жена
- брат
- сестра
- отец
- мать
- сын
- дочь
- наставник
- ученик
- член-группы
- знакомый
- подозреваемый
- жертва
- патрон
- подчинённый
- связан-с
<!-- /schema:enum:relation:kind -->
```

- [ ] **Step 3: Add Open Questions section at the end of the document**

Append after the "## Naming conventions" section:

```markdown
## Open questions policy

Entity frontmatter may carry an `open_questions:` list. Wiki-wide questions live in `questions.md`.

Entry schema (both entity-scoped and wiki-wide):

```yaml
- text: "<human-readable question>"
  category: clarify | lore | blocker
  asked: <ISO-date>
  asked_in: <session-slug>   # optional
  answer: null | "<text>"    # null while open
  answered_in: <session-slug>  # optional, set by --answer
  id: q-<tag>-NNN            # ONLY for wiki-wide questions.md entries
```

Entity-scoped entries are addressed by 0-based index. Wiki-wide entries require a unique `id:`.

When an entry is answered, the agent moves the answer into the relevant entity body (section `## Известные факты` or equivalent), removes the frontmatter entry, and bumps `updated:`. Closed entries are recovered from git history if needed — not kept in frontmatter.
```

- [ ] **Step 4: Verify SCHEMA.md still parses with the existing parser**

Run:
```bash
cd /Users/claw/.openclaw/skills/dnd && python3 -c "
import sys
sys.path.insert(0, '.')
from lib.kb_common import parse_schema
from pathlib import Path
schema = parse_schema(Path('/Users/claw/PROJECTS/dnd-wiki'))
print('types:', schema['types'])
print('enums count:', len(schema['enums']))
print('required types:', list(schema['required']))
print('taxonomy cats:', list(schema['taxonomy']))
print('OK')
"
```

Expected output includes:
- `types: ['npc', 'quest', 'location', 'session', 'character', 'item', 'rule', 'rumour']`
- `enums count: 11` (was 8, added item:type_item, character:subclass_status, open_question:category, open_question:status, relation:kind — but 3 existing categories were already partially there; exact count depends on how blocks are counted; verify >= 10)
- `OK` printed

- [ ] **Step 5: Run existing lint to confirm no regression**

Run:
```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py --strict
```

Expected: `errors=0, warnings=0` (info messages about taxonomy-hygiene are OK). Exit code 0.

- [ ] **Step 6: Commit in data repo**

```bash
cd /Users/claw/PROJECTS/dnd-wiki
git add SCHEMA.md
git commit -m "$(cat <<'EOF'
schema: document all in-use optional fields and new enums

Adds machine-readable blocks for previously-undocumented fields
(next_session, deadline, blocker, hp_max, subclass, portrait, map,
first_seen, last_seen, aliases, open_questions, active_quest_ref,
etc) and new enums (item.type_item, character.subclass_status,
open_question.category/status, relation.kind).

Non-breaking — lint does not enforce these until Phase 6 adds the
schema-undeclared-fields check.

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.1

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

Expected: commit succeeds, pre-commit hook passes (lint green).

---

### Task 2: Extend `parse_schema` to expose optional fields

**Files:**
- Modify: `/Users/claw/.openclaw/skills/dnd/lib/kb_common.py:145-175`
- Modify: `/Users/claw/.openclaw/skills/dnd/tests/conftest.py` (extend MINIMAL_SCHEMA)

- [ ] **Step 1: Read current `parse_schema` to understand structure**

Read `/Users/claw/.openclaw/skills/dnd/lib/kb_common.py` lines 145-175. Confirm `parse_schema` returns dict with keys `types, enums, taxonomy, required`. We need to add `optional`.

- [ ] **Step 2: Write failing test for optional-fields parsing**

Create/modify `/Users/claw/.openclaw/skills/dnd/tests/test_parse_schema_optional.py`:

```python
"""Tests for parse_schema handling of optional-field blocks."""
from __future__ import annotations
import textwrap
from pathlib import Path
import sys

SKILL_ROOT = Path(__file__).resolve().parent.parent
sys.path.insert(0, str(SKILL_ROOT))
from lib.kb_common import parse_schema


def test_parse_schema_returns_optional_dict(tmp_path: Path):
    schema_md = tmp_path / "SCHEMA.md"
    schema_md.write_text(textwrap.dedent("""\
        # Schema

        <!-- schema:types -->
        - npc
        - quest
        <!-- /schema:types -->

        <!-- schema:required:npc -->
        - status
        <!-- /schema:required:npc -->

        <!-- schema:optional:npc -->
        - first_seen
        - last_seen
        <!-- /schema:optional:npc -->

        <!-- schema:optional:common -->
        - aliases
        - portrait
        <!-- /schema:optional:common -->
    """))
    result = parse_schema(tmp_path)
    assert "optional" in result
    assert result["optional"]["npc"] == ["first_seen", "last_seen"]
    assert result["optional"]["common"] == ["aliases", "portrait"]


def test_parse_schema_optional_absent_returns_empty_dict(tmp_path: Path):
    schema_md = tmp_path / "SCHEMA.md"
    schema_md.write_text("# Schema\n\n<!-- schema:types -->\n- npc\n<!-- /schema:types -->\n")
    result = parse_schema(tmp_path)
    assert result.get("optional", {}) == {}
```

- [ ] **Step 3: Run test, expect failure**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest tests/test_parse_schema_optional.py -v
```

Expected: FAIL on `test_parse_schema_returns_optional_dict` (KeyError or missing "optional" key).

- [ ] **Step 4: Implement — modify `parse_schema` in `lib/kb_common.py`**

Locate the existing function near line 145. Replace the `result` initialization and loop body:

```python
def parse_schema(dnd_dir: Path) -> dict:
    """Parse SCHEMA.md machine-readable blocks.

    Returns {types, enums, taxonomy, required, optional} mappings.
    """
    schema_path = dnd_dir / "SCHEMA.md"
    if not schema_path.exists():
        raise SystemExit(f"SCHEMA.md missing at {schema_path}")
    text = schema_path.read_text(encoding="utf-8")
    result: dict = {
        "types": [],
        "enums": {},
        "taxonomy": {},
        "required": {},
        "optional": {},
    }
    for m in _BLOCK_RE.finditer(text):
        block_id = m.group("id")
        body = m.group("body")
        parts = block_id.split(":")
        if parts[0] == "types":
            result["types"] = _parse_list(body)
        elif parts[0] == "enum" and len(parts) == 3:
            _, type_name, field = parts
            result["enums"].setdefault(type_name, {})[field] = _parse_list(body)
        elif parts[0] == "taxonomy":
            result["taxonomy"] = _parse_yaml_like(body)
        elif parts[0] == "required" and len(parts) == 2:
            _, type_name = parts
            result["required"][type_name] = _parse_list(body)
        elif parts[0] == "optional" and len(parts) == 2:
            _, type_name = parts
            result["optional"][type_name] = _parse_list(body)
    return result
```

- [ ] **Step 5: Run test, expect pass**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest tests/test_parse_schema_optional.py -v
```

Expected: both tests PASS.

- [ ] **Step 6: Run full existing test suite to confirm no regression**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest
```

Expected: all tests pass (previous count + 2 new).

- [ ] **Step 7: Commit skill repo**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/lib/kb_common.py skills/dnd/tests/test_parse_schema_optional.py
git commit -m "$(cat <<'EOF'
feat(schema): parse_schema exposes optional-fields blocks

parse_schema now returns dict with 'optional' key in addition to
'types', 'enums', 'taxonomy', 'required'. Backward-compatible — existing
callers that only read types/enums/required/taxonomy keep working.

Required by schema-undeclared-fields lint check (Phase 6 L9).

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Phase 2: Relations unification

Phase goal: one canonical shape — `relations: [{to: <slug>, kind: <enum>}]`. Extend `_iter_refs` to recognize both old and new shape during migration (backward-compatible). Write a migration script and run it.

### Task 3: Extend `_iter_refs` to accept unified `{to, kind}` shape

**Files:**
- Modify: `/Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py:200-236`
- Modify: `/Users/claw/.openclaw/skills/dnd/tests/test_dnd_lint_new.py` (add tests)

- [ ] **Step 1: Write failing test for unified relations shape**

Append to `tests/test_dnd_lint_new.py`:

```python
def test_broken_ref_unified_relations_shape(wiki):
    """relations: [{to: slug, kind: str}] — broken-ref should detect bad slug."""
    (wiki / "npcs" / "testnpc.md").write_text(
        "---\n"
        "name: Test NPC\n"
        "slug: testnpc\n"
        "type: npc\n"
        "race: Человек\n"
        "role: test\n"
        "location: test-loc\n"
        "status: живой\n"
        "relation: союзник\n"
        "relations:\n"
        "  - to: nonexistent-slug\n"
        "    kind: союзник\n"
        "tags: []\n"
        "sources: []\n"
        "created: 2026-04-24\n"
        "updated: 2026-04-24\n"
        "---\n\nbody\n"
    )
    data = run_lint(wiki, "--only", "broken-ref")
    assert any(
        i["check"] == "broken-ref" and "nonexistent-slug" in i["message"]
        for i in data["issues"]
    ), data


def test_broken_ref_unified_relations_ok(wiki):
    """relations: [{to: slug, kind: str}] with valid slug — no error."""
    (wiki / "npcs" / "alpha.md").write_text(
        "---\nname: Alpha\nslug: alpha\ntype: npc\nrace: x\nrole: x\n"
        "location: test-loc\nstatus: живой\nrelation: нейтральный\n"
        "tags: []\nsources: []\ncreated: 2026-04-24\nupdated: 2026-04-24\n---\n\nx\n"
    )
    (wiki / "npcs" / "beta.md").write_text(
        "---\nname: Beta\nslug: beta\ntype: npc\nrace: x\nrole: x\n"
        "location: test-loc\nstatus: живой\nrelation: союзник\n"
        "relations:\n  - to: alpha\n    kind: союзник\n"
        "tags: []\nsources: []\ncreated: 2026-04-24\nupdated: 2026-04-24\n---\n\nx\n"
    )
    data = run_lint(wiki, "--only", "broken-ref")
    assert not any(i["check"] == "broken-ref" for i in data["issues"]), data
```

- [ ] **Step 2: Run test, expect failure**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest tests/test_dnd_lint_new.py::test_broken_ref_unified_relations_shape tests/test_dnd_lint_new.py::test_broken_ref_unified_relations_ok -v
```

Expected: `test_broken_ref_unified_relations_shape` FAILS (old `_iter_refs` does not yield `to:` key entries).

- [ ] **Step 3: Modify `_iter_refs` in `dnd-lint.py`**

Locate `_iter_refs` near line 200. Replace the loop that handles `list` shape of relations (currently around lines 217-229) with this:

```python
    elif isinstance(rels, list):
        for item in rels:
            if not isinstance(item, dict):
                continue
            # Unified shape: {to: slug, kind: str}
            if "to" in item and isinstance(item["to"], str) and item["to"]:
                yield "relations.to", item["to"]
                continue
            # Legacy shapes (pre-v2): {npc: slug, type: str}, {location: slug, ...}
            for rfld, rval in item.items():
                if rfld in ("type", "kind"):
                    continue  # relation-kind label, not a slug
                if isinstance(rval, str) and rval:
                    yield f"relations.{rfld}", rval
                elif isinstance(rval, list):
                    for v in rval:
                        if isinstance(v, str):
                            yield f"relations.{rfld}", v
```

- [ ] **Step 4: Run both new tests, expect pass**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest tests/test_dnd_lint_new.py::test_broken_ref_unified_relations_shape tests/test_dnd_lint_new.py::test_broken_ref_unified_relations_ok -v
```

Expected: both PASS.

- [ ] **Step 5: Run full test suite to confirm no regression**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest
```

Expected: all tests pass. Old tests that use legacy `{npc: slug, type: str}` shape continue to work (fallback loop).

- [ ] **Step 6: Commit**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/scripts/dnd-lint.py skills/dnd/tests/test_dnd_lint_new.py
git commit -m "$(cat <<'EOF'
feat(lint): _iter_refs accepts unified relations shape {to, kind}

Extends broken-ref check to recognize the canonical v2 relations
format while retaining backward-compat for legacy {npc, type} shape
during migration. No data changes yet.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 4: Write migration script `migrate_relations_v2.py`

**Files:**
- Create: `/Users/claw/.openclaw/skills/dnd/scripts/migrate_relations_v2.py`
- Create: `/Users/claw/.openclaw/skills/dnd/tests/test_migrate_relations_v2.py`

- [ ] **Step 1: Write failing tests for migration logic**

Create `tests/test_migrate_relations_v2.py`:

```python
"""Tests for one-time relations migration to unified {to, kind} shape."""
from __future__ import annotations
import importlib.util
import sys
from pathlib import Path

SKILL_ROOT = Path(__file__).resolve().parent.parent
sys.path.insert(0, str(SKILL_ROOT))

spec = importlib.util.spec_from_file_location(
    "migrate_relations_v2",
    SKILL_ROOT / "scripts" / "migrate_relations_v2.py",
)
mod = importlib.util.module_from_spec(spec)
spec.loader.exec_module(mod)


def test_convert_legacy_npc_type_list():
    legacy = [{"npc": "liana", "type": "союзник"}, {"npc": "eymar", "type": "враг"}]
    assert mod.convert_relations(legacy) == [
        {"to": "liana", "kind": "союзник"},
        {"to": "eymar", "kind": "враг"},
    ]


def test_convert_legacy_with_descriptive_type():
    legacy = [{"npc": "lander", "type": "коллега (защитник того же храма)"}]
    # Descriptive parenthetical stripped, kind matched to enum best-fit
    assert mod.convert_relations(legacy) == [
        {"to": "lander", "kind": "коллега"}
    ]


def test_convert_dict_shape_relations():
    # relations: {allies: [slug1, slug2], enemies: [slug3]}
    legacy = {"allies": ["alpha", "beta"], "enemies": ["gamma"]}
    assert mod.convert_relations(legacy) == [
        {"to": "alpha", "kind": "союзник"},
        {"to": "beta", "kind": "союзник"},
        {"to": "gamma", "kind": "враг"},
    ]


def test_already_unified_passthrough():
    already = [{"to": "alpha", "kind": "союзник"}]
    assert mod.convert_relations(already) == already


def test_unknown_kind_falls_back_to_connected():
    legacy = [{"npc": "x", "type": "странное описание"}]
    assert mod.convert_relations(legacy) == [{"to": "x", "kind": "связан-с"}]


def test_migrate_file_writes_updated_frontmatter(tmp_path: Path):
    # full file round-trip
    from lib.kb_common import read_frontmatter, write_frontmatter
    p = tmp_path / "test.md"
    write_frontmatter(
        p,
        {
            "name": "T",
            "slug": "t",
            "type": "npc",
            "race": "x",
            "role": "x",
            "location": "l",
            "status": "живой",
            "relation": "нейтральный",
            "tags": [],
            "sources": [],
            "created": "2026-04-24",
            "updated": "2026-04-24",
            "relations": [{"npc": "alpha", "type": "союзник"}],
        },
        "body\n",
    )
    mod.migrate_file(p)
    meta, _ = read_frontmatter(p)
    assert meta["relations"] == [{"to": "alpha", "kind": "союзник"}]
```

- [ ] **Step 2: Run test, expect failure (module does not exist)**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest tests/test_migrate_relations_v2.py -v
```

Expected: FAIL with ModuleNotFoundError or FileNotFoundError.

- [ ] **Step 3: Implement `scripts/migrate_relations_v2.py`**

Create the file:

```python
#!/usr/bin/env python3
"""One-time migration: convert relations to unified {to, kind} shape.

Reads every *.md in entity folders (npcs, quests, locations, sessions,
characters, items, rules). If the file has a `relations:` field in legacy
shape (list of dicts with `npc:`/`location:`/other slug keys, OR flat dict
with allies/enemies/etc), rewrites it to:

    relations:
      - to: <slug>
        kind: <relation.kind enum>

Idempotent: running twice is a no-op.
"""
from __future__ import annotations
import argparse
import re
import sys
from pathlib import Path

SCRIPT_DIR = Path(__file__).resolve().parent
sys.path.insert(0, str(SCRIPT_DIR.parent))
from lib.kb_common import (  # noqa: E402
    resolve_config, read_frontmatter, write_frontmatter, append_log,
)

TYPE_FOLDERS = ["npcs", "quests", "locations", "sessions", "characters", "items", "rules"]

# Map legacy descriptive types → canonical relation.kind enum values.
KIND_KEYWORDS = [
    ("союзник", "союзник"),
    ("враг", "враг"),
    ("нейтрал", "нейтральный"),
    ("коллега", "коллега"),
    ("муж", "муж"),
    ("жена", "жена"),
    ("брат", "брат"),
    ("сестра", "сестра"),
    ("отец", "отец"),
    ("мать", "мать"),
    ("сын", "сын"),
    ("дочь", "дочь"),
    ("наставник", "наставник"),
    ("ученик", "ученик"),
    ("член", "член-группы"),
    ("подозрева", "подозреваемый"),
    ("жертва", "жертва"),
    ("патрон", "патрон"),
    ("подчин", "подчинённый"),
    ("знаком", "знакомый"),
]

DICT_KEY_TO_KIND = {
    "allies": "союзник",
    "enemies": "враг",
    "family": "член-группы",
    "colleagues": "коллега",
    "mentor_of": "наставник",
    "student_of": "ученик",
}

SLUG_KEYS = {"npc", "location", "character", "quest", "item"}


def _kind_from_descriptive(raw: str) -> str:
    """Pick the best-matching kind enum for a descriptive string."""
    if not isinstance(raw, str) or not raw.strip():
        return "связан-с"
    lower = raw.lower()
    for needle, kind in KIND_KEYWORDS:
        if needle in lower:
            return kind
    return "связан-с"


def convert_relations(rels) -> list[dict]:
    """Normalize any legacy shape to list[{to, kind}]."""
    if rels is None:
        return []
    # Already unified?
    if isinstance(rels, list) and rels and all(
        isinstance(r, dict) and set(r.keys()) == {"to", "kind"} for r in rels
    ):
        return list(rels)
    out: list[dict] = []
    if isinstance(rels, dict):
        # Flat shape: {allies: [slug], enemies: [slug]}
        for key, kind in DICT_KEY_TO_KIND.items():
            for slug in rels.get(key, []) or []:
                if isinstance(slug, str) and slug:
                    out.append({"to": slug, "kind": kind})
        return out
    if not isinstance(rels, list):
        return []
    for item in rels:
        if not isinstance(item, dict):
            continue
        if "to" in item and "kind" in item:
            out.append({"to": item["to"], "kind": item["kind"]})
            continue
        # Legacy {npc/location/etc: slug, type: "descriptive"}
        slug = None
        for k in SLUG_KEYS:
            if k in item and isinstance(item[k], str) and item[k]:
                slug = item[k]
                break
        if slug is None:
            continue
        raw_kind = item.get("type") or item.get("kind") or ""
        out.append({"to": slug, "kind": _kind_from_descriptive(raw_kind)})
    return out


def migrate_file(path: Path) -> bool:
    meta, body = read_frontmatter(path)
    if not meta or "relations" not in meta:
        return False
    new_rels = convert_relations(meta["relations"])
    if new_rels == meta["relations"]:
        return False
    meta["relations"] = new_rels
    write_frontmatter(path, meta, body)
    return True


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--dry-run", action="store_true")
    args = parser.parse_args()
    cfg = resolve_config()
    dnd_dir = cfg["dnd_dir"]
    changed = []
    for folder in TYPE_FOLDERS:
        d = dnd_dir / folder
        if not d.exists():
            continue
        for p in sorted(d.glob("*.md")):
            if p.name.startswith("_"):
                continue
            if args.dry_run:
                meta, _ = read_frontmatter(p)
                if meta and "relations" in meta:
                    new = convert_relations(meta["relations"])
                    if new != meta["relations"]:
                        changed.append(p)
            else:
                if migrate_file(p):
                    changed.append(p)
    print(f"{'Would migrate' if args.dry_run else 'Migrated'}: {len(changed)} files")
    for p in changed:
        print(f"  {p.relative_to(dnd_dir)}")
    if not args.dry_run:
        append_log(dnd_dir, "migrate", "relations-to-v2",
                   [f"- {p.relative_to(dnd_dir)}" for p in changed])


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Run tests, expect pass**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest tests/test_migrate_relations_v2.py -v
```

Expected: all 6 tests PASS.

- [ ] **Step 5: Commit migration script**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/scripts/migrate_relations_v2.py skills/dnd/tests/test_migrate_relations_v2.py
git commit -m "$(cat <<'EOF'
feat(migrate): one-time relations v2 normalization script

Converts legacy relations shapes (list of {npc, type}, dict of
allies/enemies, etc) to canonical [{to, kind}]. Idempotent.
Not yet invoked against real data — Task 5 runs it.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 5: Run relations migration against real wiki

**Files:**
- Modify: every entity file under `/Users/claw/PROJECTS/dnd-wiki/{npcs,quests,locations,sessions,characters,items,rules}/*.md` that has legacy relations

- [ ] **Step 1: Dry-run to see what will change**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/migrate_relations_v2.py --dry-run
```

Expected: list of files with legacy relations. Known candidates from audit: `npcs/liana.md`, `npcs/rouen.md`, `npcs/michael.md`, `npcs/ldap.md`, `npcs/rbak.md`, `npcs/berser-trant.md` (if it has any), plus several quests with `related_npcs` (those are a separate optional field, NOT migrated here — only `relations:` key).

- [ ] **Step 2: Apply migration**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/migrate_relations_v2.py
```

Expected: "Migrated: N files" where N > 0; each file listed.

- [ ] **Step 3: Spot-check output**

```bash
grep -A 3 "^relations:" /Users/claw/PROJECTS/dnd-wiki/npcs/liana.md
grep -A 3 "^relations:" /Users/claw/PROJECTS/dnd-wiki/npcs/rouen.md
grep -A 3 "^relations:" /Users/claw/PROJECTS/dnd-wiki/npcs/michael.md
```

Expected: each shows `relations:` followed by `- to: <slug>` and `kind: <enum>` lines; no `npc:` or `type:` keys.

- [ ] **Step 4: Run lint to confirm green**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py --strict
```

Expected: `errors=0, warnings=0` (info only).

- [ ] **Step 5: Commit in data repo**

```bash
cd /Users/claw/PROJECTS/dnd-wiki
git add .
git commit -m "$(cat <<'EOF'
schema: unify relations format across all entities (v2)

All relations: blocks now use canonical [{to: <slug>, kind: <enum>}]
shape. Legacy {npc, type} and dict-of-lists shapes removed.

Run via scripts/migrate_relations_v2.py (idempotent).

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.1

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Phase 3: Data fixes batch

Phase goal: resolve the 17 verified data defects from audit §1. All in a single commit `fix: data integrity — 17 issues` at the end of the phase.

### Task 6: Fix quest-level errors (main-luskan, ship-irlin-forest, stone-bee, statya-irvin)

**Files:**
- Modify: `/Users/claw/PROJECTS/dnd-wiki/quests/main-luskan.md`
- Modify: `/Users/claw/PROJECTS/dnd-wiki/quests/ship-irlin-forest.md`
- Modify: `/Users/claw/PROJECTS/dnd-wiki/quests/stone-bee.md`
- Modify: `/Users/claw/PROJECTS/dnd-wiki/quests/statya-irvin.md`

- [ ] **Step 1: Read each file, note current frontmatter**

```bash
for f in main-luskan ship-irlin-forest stone-bee statya-irvin; do
    echo "=== $f ==="
    head -20 /Users/claw/PROJECTS/dnd-wiki/quests/$f.md
done
```

- [ ] **Step 2: Edit `main-luskan.md` — giver becomes null, add open_question**

Use Edit on `/Users/claw/PROJECTS/dnd-wiki/quests/main-luskan.md`. Change:

```yaml
giver: '2026-01-06'
```

to:

```yaml
giver: null
open_questions:
  - text: "Кто formally дал квест «Путь в Лускан»? NPC-торговец из сессии 1, или общая цель партии?"
    category: clarify
    asked: 2026-04-24
    asked_in: 2026-01-06
    answer: null
```

Also bump `updated: '2026-04-24'` if needed (no-op if already current).

- [ ] **Step 3: Edit `ship-irlin-forest.md` — location slug fix**

Change:

```yaml
location: Ирлинский лес
```

to:

```yaml
location: irlin-forest
```

- [ ] **Step 4: Edit `stone-bee.md` — location fill from null**

Change:

```yaml
location: null
```

to:

```yaml
location: northern-keller
```

(best inference per audit; if the dev later learns it should be coast-related, change is trivial).

- [ ] **Step 5: Edit `statya-irvin.md` — priority on completed quest**

Current has `status: выполнен` + `priority: срочно`. Change `priority: срочно` to:

```yaml
priority: побочный
```

(Valid enum value; completed-побочный is a harmless combination.)

- [ ] **Step 6: Verify lint still green**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py --strict
```

Expected: errors=0, warnings=0.

- [ ] **Step 7: (No commit yet — batched at end of Phase 3 Task 9)**

---

### Task 7: Fix character placeholders (hp_max for 4 PC, subclass_status for 2 PC)

**Files:**
- Modify: `/Users/claw/PROJECTS/dnd-wiki/characters/ashan.md`
- Modify: `/Users/claw/PROJECTS/dnd-wiki/characters/chad-ridley.md`
- Modify: `/Users/claw/PROJECTS/dnd-wiki/characters/ironiya-leonberger.md`
- Modify: `/Users/claw/PROJECTS/dnd-wiki/characters/dimitrina-speiskaya.md`
- Modify: `/Users/claw/PROJECTS/dnd-wiki/characters/torvin-kamneklyatv.md`

- [ ] **Step 1: Add `hp_max: null` + open_question to the 4 PC without it**

For each of ashan, chad-ridley, ironiya-leonberger, dimitrina-speiskaya — Edit the frontmatter to insert between existing fields:

```yaml
hp_max: null
```

Then add an `open_questions:` entry (either append if the key already exists from Task 6, or create new list):

```yaml
open_questions:
  - text: "HP max у персонажа — посмотреть в character sheet"
    category: clarify
    asked: 2026-04-24
    answer: null
```

- [ ] **Step 2: Convert subclass placeholders for dimitrina, ironiya**

Edit `characters/dimitrina-speiskaya.md` frontmatter:

```yaml
subclass: не выбрана
```

replace with:

```yaml
subclass: null
subclass_status: tbd
```

Also add open_question:

```yaml
  - text: "Субкласс Димитрины (плут) — выбрать"
    category: clarify
    asked: 2026-04-24
    answer: null
```

Repeat analogous edit for `ironiya-leonberger.md` — replace `subclass: уточнить` with `subclass: null` + `subclass_status: tbd` + open_question.

- [ ] **Step 3: For the 3 PCs with chosen subclass (ashan, chad-ridley, torvin-kamneklyatv) add `subclass_status: chosen`**

Edit each of ashan.md, chad-ridley.md, torvin-kamneklyatv.md — insert after the existing `subclass:` line:

```yaml
subclass_status: chosen
```

- [ ] **Step 4: Add `active_quest_ref: cure-mummy-rot` to Torvin**

Edit `characters/torvin-kamneklyatv.md`, insert in frontmatter:

```yaml
active_quest_ref: cure-mummy-rot
```

- [ ] **Step 5: Verify lint green**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py --strict
```

Expected: errors=0, warnings=0. (`subclass_status` and `hp_max` and `open_questions` are now all declared optional in SCHEMA as of Phase 1.)

---

### Task 8: Remove `secrets:` field from berser-trant, relocate asset, delete `_party.md`

**Files:**
- Modify: `/Users/claw/PROJECTS/dnd-wiki/npcs/berser-trant.md`
- Move: `/Users/claw/PROJECTS/dnd-wiki/npcs/mystery-mage.jpg` → `/Users/claw/PROJECTS/dnd-wiki/assets/npcs/mystery-mage.jpg`
- Modify: `/Users/claw/PROJECTS/dnd-wiki/npcs/eymar.md` (body link update)
- Modify: `/Users/claw/PROJECTS/dnd-wiki/characters/chad-ridley.md` (receive note)
- Delete: `/Users/claw/PROJECTS/dnd-wiki/characters/_party.md`

- [ ] **Step 1: Remove `secrets:` from `berser-trant.md`, migrate content to body**

Read `/Users/claw/PROJECTS/dnd-wiki/npcs/berser-trant.md`. Locate:

```yaml
secrets:
- Тайный участник «Братства взирающих очей»
```

Delete those two lines.

Add to `tags:` list: `братство-взирающих-очей` (already present in `factions` taxonomy).

In body, after `## Роль` section, add:

```markdown
## Тайная сторона

Берсер — тайный участник культа **Братство взирающих очей**. Партия узнала об этом из местной газеты Северного Келлера (сессия 2). Для большинства жителей он остаётся публичной фигурой — «Добродетелью».
```

- [ ] **Step 2: Relocate asset file**

```bash
mkdir -p /Users/claw/PROJECTS/dnd-wiki/assets/npcs
mv /Users/claw/PROJECTS/dnd-wiki/npcs/mystery-mage.jpg /Users/claw/PROJECTS/dnd-wiki/assets/npcs/mystery-mage.jpg
```

- [ ] **Step 3: Update `eymar.md` body reference**

Read `/Users/claw/PROJECTS/dnd-wiki/npcs/eymar.md`. Change:

```markdown
- Изображение: `mystery-mage.jpg`
```

to:

```markdown
- Изображение: `assets/npcs/mystery-mage.jpg`
```

- [ ] **Step 4: Move `_party.md` note about Коля into `chad-ridley.md` body**

Read current `/Users/claw/PROJECTS/dnd-wiki/characters/_party.md`, find:

```markdown
**Коля:** 14 февраля 2026 психанул и ливнул из чата (и из чата Цивы). Статус участия в кампании неясен.
```

Edit `/Users/claw/PROJECTS/dnd-wiki/characters/chad-ridley.md`, add to body (after the `## Бэкстори` or equivalent section, or at end before open_questions):

```markdown
## Заметки игрока

- 14 февраля 2026 (сессия #3) — Коля психанул и ливнул из чата (и из чата Цивы). Статус участия в кампании неясен.
```

Analogously move the Daха note (ГМ солгал во 2й сессии) into `characters/ashan.md` body as a note section. Move the Ринвасильна note (любит когда называют полным именем) into `characters/ironiya-leonberger.md`.

- [ ] **Step 5: Delete `_party.md`**

```bash
rm /Users/claw/PROJECTS/dnd-wiki/characters/_party.md
```

- [ ] **Step 6: Verify lint green**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py --strict
```

Expected: errors=0, warnings=0.

- [ ] **Step 7: (Still no commit — batched at end of Phase 3 Task 9)**

---

### Task 9: Commit data fixes batch

**Files:** All modifications from Tasks 6-8.

- [ ] **Step 1: Review all pending changes**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && git status && echo "---" && git diff --stat
```

Expected: modifications across `quests/`, `characters/`, `npcs/`, move+delete in assets/npcs/, deletion of `characters/_party.md`.

- [ ] **Step 2: Stage all and commit**

```bash
cd /Users/claw/PROJECTS/dnd-wiki
git add -A quests/ characters/ npcs/ assets/ log.md
git rm characters/_party.md 2>/dev/null || true
git commit -m "$(cat <<'EOF'
fix: data integrity — 17 issues from audit

Quests:
- main-luskan: giver date → null + open_question
- ship-irlin-forest: location Cyrillic → irlin-forest
- stone-bee: location null → northern-keller
- statya-irvin: priority срочно → побочный (quest is выполнен)

Characters:
- hp_max added (null + open_question) for 4 PC
- subclass: "не выбрана"/"уточнить" → null + subclass_status: tbd
  + open_question (dimitrina, ironiya)
- subclass_status: chosen on ashan, chad-ridley, torvin-kamneklyatv
- active_quest_ref: cure-mummy-rot on torvin-kamneklyatv

NPCs:
- berser-trant: secrets field → body section + tag
- mystery-mage.jpg → assets/npcs/; eymar.md path updated
- _party.md deleted; notes migrated to respective character bodies

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.6

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

Expected: commit succeeds, pre-commit hook green.

---

## Phase 4: Taxonomy cleanup

Phase goal: reorganize SCHEMA.md taxonomy into semantically-coherent groups; migrate tags on all entity files.

### Task 10: Reorganize SCHEMA.md taxonomy block

**Files:**
- Modify: `/Users/claw/PROJECTS/dnd-wiki/SCHEMA.md`

- [ ] **Step 1: Read current taxonomy block**

```bash
sed -n '/<!-- schema:taxonomy -->/,/<!-- \/schema:taxonomy -->/p' /Users/claw/PROJECTS/dnd-wiki/SCHEMA.md
```

- [ ] **Step 2: Replace taxonomy block with reorganized version**

Use Edit on `/Users/claw/PROJECTS/dnd-wiki/SCHEMA.md`. Replace the entire `<!-- schema:taxonomy --> ... <!-- /schema:taxonomy -->` block with:

```
<!-- schema:taxonomy -->
regions: [лускан, побережье-меча, побережье, острова-ирлин, остров-болото, материк, лес, ирлинский-лес, природа, водопад, болота]
settlements: [северный-келлер, южный-келлер, город-угасания, даггерфорд]
buildings: [таверна, голодный-лесник, горный-лесник, чайная, дом-прессы, библиотека, академия-магии, арена]
shrines: [храм-ильматера, алтарь, святилище]
dungeons: [подземелье, катакомбы, склеп, руины, кемп-орков, заброшенная-лесопилка, лаборатория]
landmarks: [ферма-самблео, дерево-илкай, обелиск, жаровня]
factions: [академия-огненные-ладони, братство-взирающих-очей, орден-каменной-клятвы, круг-луны, семья-файрмей, братья-бись, группа-приключенцев, первая-группа, огненные-ладони]
races: [дворф, эльф, человек, тифлинг, полурослик, дракон-рождённый]
classes: [паладин, друид, маг, колдун, плут, клерик, воин, варвар]
status-markers: [активный, заморожен, проклятие, срочно, главный-квест, побочный, выполнен, мёртв, погиб, квест, основная-локация, цель, враг]
mechanics: [mummy-rot, договор-души, обелиски, жаровни, ментальная-атака, электричество, перемещение, механика, демиплан, крушение, самовзрыв, побег, прорицание, опознание, ваншот, апгрейд, клятва-мести]
concepts: [катастрас, лабиринт, мистерия, выход-с-острова, карта, зарисовка]
creatures: [призрак, нежить, мумия, гомункул, кракен, мимик, питомец, шершни, шершень-символ, орки, гоблины]
items: [амулет, кольцо, волынка, ячейки, книга-заклинаний, камень, камень-с-шершнем, записка, письмо, письмо-женевьевы, воздушный-шар, гейзенберг, алигары, кресло, повозка]
npcs-named: [роуан-файрмей, бабушка-любава, гидеон, мануэль, лиана, миала, эймар, маргарита, катрин, торвин, ашен, женевьева, самблео, линда-чит, лала-эншейн, берсер-трант, алистер, ильматер]
roles: [профессор, капитан, журналист, мэр, трактирщик, проводник, жрица, пасечник, прорицательница, торговец-информацией, исследователь, защитник, ученик, подозреваемый, подозрительная, беременная, любопытная, важный, добродетель, антагонист, загадочный-маг, жертва, торговцы, власть]
events: [скандал, статья, слух, выставка, аукцион, выкуп, путешествие, проход, огненный-круг, корабль, торговля]
<!-- /schema:taxonomy -->
```

Key changes:
- `cities:` renamed and SPLIT into `settlements`, `buildings`, `shrines`, `dungeons`, `landmarks`
- `events:` kept, but removed `все-локации` (meta tag — useless), `катастрас` moved to `concepts:`, `лабиринт` moved to `concepts:`
- `creatures:` removed `призраки` (kept `призрак`) — de-duplication
- Added to `factions:` the alias `огненные-ладони` (matched existing tag usage)
- Added to `settlements:` : `даггерфорд` (referenced in timeline but never taxonomized)
- New category `concepts:` for abstract narrative concepts

- [ ] **Step 3: Run lint, confirm no new errors**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py --strict
```

Expected result: **warnings will appear** on every entity whose `tags:` reference a tag that moved category but still exists (`lint.taxonomy` does not care about category, only membership). So warnings should stay 0. But tags that were REMOVED (`все-локации`, `призраки`) will trigger warnings.

If `таксономия` check fails for removed tags — this is expected; Task 11 fixes them.

```
Summary: errors=0, warnings=N, info=...
```

Where N = count of pages with removed-or-renamed tags. Note N and move on.

- [ ] **Step 4: (No commit yet — Task 11 fixes the tag references)**

---

### Task 11: Migrate entity tags to match reorganized taxonomy

**Files:**
- Modify: every file under `npcs/`, `quests/`, `locations/`, `sessions/`, `characters/`, `items/`, `rules/` that has the removed tags `все-локации`, `призраки`

- [ ] **Step 1: Find all pages using removed tags**

```bash
grep -rln "все-локации\|призраки" /Users/claw/PROJECTS/dnd-wiki/{npcs,quests,locations,sessions,characters,items,rules} 2>/dev/null
```

- [ ] **Step 2: For each file, edit tags**

For `- все-локации` tags: remove the line entirely (it was a meta-tag).

For `- призраки` tags: replace with `- призрак` (singular form).

Use Edit per file; example for one:

```python
# (Pattern for the agent executing this task — real Edit per file)
Edit(file_path=..., old_string="- призраки\n", new_string="- призрак\n")
```

If a file has BOTH `- призрак` and `- призраки` after edit, leave only one (dedupe).

- [ ] **Step 3: Run lint, confirm green**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py --strict
```

Expected: errors=0, warnings=0.

- [ ] **Step 4: Commit taxonomy reorganization + tag migration**

```bash
cd /Users/claw/PROJECTS/dnd-wiki
git add SCHEMA.md npcs/ quests/ locations/ sessions/ characters/ items/ rules/ log.md
git commit -m "$(cat <<'EOF'
schema: reorganize taxonomy into semantic groups

- cities: → split into settlements/buildings/shrines/dungeons/landmarks
- events: → stripped; narrative concepts moved to new concepts: category
- creatures: → removed призраки (kept призрак); dedup
- Added даггерфорд to settlements
- Added огненные-ладони alias to factions
- Migrated entity tags to match reorganized taxonomy

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.2

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Phase 5: Open questions infrastructure

Phase goal: `questions.md` + `dnd-questions.py` CLI + L7 (unresolved-placeholders) + L8 (stale-open-question).

### Task 12: Create `questions.md` scaffold

**Files:**
- Create: `/Users/claw/PROJECTS/dnd-wiki/questions.md`

- [ ] **Step 1: Write initial questions.md with seed entries**

Create `/Users/claw/PROJECTS/dnd-wiki/questions.md`:

```markdown
# Вопросы ДМу — The story of Irlin

*Вики-wide вопросы к ГМу. Не привязаны к конкретной entity. Entity-scoped вопросы — в frontmatter соответствующей страницы (`open_questions:`).*

*Статусы: open | answered | obsolete. Категории: clarify | lore | blocker. ID формат: `q-<tag>-NNN`.*

---

```yaml
- id: q-cal-001
  text: "Почему STATE.date_ingame=день ~47, а timeline.md последняя сессия (2026-03-28) заканчивается днём 17? Off-screen время между сессиями? Опечатка?"
  category: clarify
  asked: 2026-04-24
  answer: null

- id: q-lor-001
  text: "Точная дата раскола Ирлина — когда именно? В timeline стоит ~900+"
  category: lore
  asked: 2026-04-24
  answer: null

- id: q-lor-002
  text: "Точные даты событий с первой группой приключенцев (Линда, Женевьева, Бастиан, Мануэль) — когда была попытка Катастраса?"
  category: lore
  asked: 2026-04-24
  answer: null
```
```

Note: the triple-backtick is nested — use the `indent-based YAML fencing` pattern consistent with `rumours.md`.

- [ ] **Step 2: Run lint to confirm parser does not choke**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py --strict
```

Expected: errors=0, warnings=0 (parser ignores unknown files).

- [ ] **Step 3: Commit**

```bash
cd /Users/claw/PROJECTS/dnd-wiki
git add questions.md log.md
git commit -m "$(cat <<'EOF'
feat: add questions.md for wiki-wide open questions

Scaffold with 3 seed questions migrated from audit:
- q-cal-001: calendar discrepancy (day 47 vs 17)
- q-lor-001: Irlin split exact date
- q-lor-002: first-group timeline precision

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.7

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 13: Implement `dnd-questions.py` CLI — part 1: list + ask

**Files:**
- Create: `/Users/claw/.openclaw/skills/dnd/scripts/dnd-questions.py`
- Create: `/Users/claw/.openclaw/skills/dnd/tests/test_dnd_questions.py`

- [ ] **Step 1: Write failing tests for list + ask commands**

Create `tests/test_dnd_questions.py`:

```python
"""Tests for dnd-questions.py CLI."""
from __future__ import annotations
import json
import os
import subprocess
import sys
import textwrap
from pathlib import Path

SKILL_ROOT = Path(__file__).resolve().parent.parent
QUESTIONS = SKILL_ROOT / "scripts" / "dnd-questions.py"


def run_q(wiki_dir: Path, *args: str, **kwargs):
    env = {**os.environ, "DND_DIR": str(wiki_dir)}
    return subprocess.run(
        [sys.executable, str(QUESTIONS), *args],
        capture_output=True, text=True, env=env, **kwargs,
    )


def test_list_empty_wiki(wiki):
    r = run_q(wiki)
    assert r.returncode == 0
    assert "нет открытых вопросов" in r.stdout.lower() or "0" in r.stdout


def test_list_entity_scoped_questions(wiki):
    # Write an NPC file with open_questions
    (wiki / "npcs" / "alpha.md").write_text(textwrap.dedent("""\
        ---
        name: Alpha
        slug: alpha
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: нейтральный
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        open_questions:
          - text: "Как выглядит Alpha?"
            category: clarify
            asked: 2026-04-24
            answer: null
        ---

        body
    """))
    r = run_q(wiki)
    assert r.returncode == 0
    assert "Как выглядит Alpha?" in r.stdout
    assert "alpha" in r.stdout  # entity slug displayed
    assert "clarify" in r.stdout


def test_list_wiki_wide_questions(wiki):
    (wiki / "questions.md").write_text(textwrap.dedent("""\
        # Questions

        ```yaml
        - id: q-tst-001
          text: "Global question?"
          category: lore
          asked: 2026-04-24
          answer: null
        ```
    """))
    r = run_q(wiki)
    assert r.returncode == 0
    assert "q-tst-001" in r.stdout
    assert "Global question?" in r.stdout


def test_by_category_filter(wiki):
    (wiki / "npcs" / "alpha.md").write_text(textwrap.dedent("""\
        ---
        name: Alpha
        slug: alpha
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: нейтральный
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        open_questions:
          - text: "A"
            category: clarify
            asked: 2026-04-24
            answer: null
          - text: "B"
            category: blocker
            asked: 2026-04-24
            answer: null
        ---

        body
    """))
    r = run_q(wiki, "--by-category", "blocker")
    assert r.returncode == 0
    assert "B" in r.stdout
    assert "A" not in r.stdout


def test_ask_adds_entity_scoped(wiki):
    (wiki / "npcs" / "alpha.md").write_text(textwrap.dedent("""\
        ---
        name: Alpha
        slug: alpha
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: нейтральный
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        ---

        body
    """))
    r = run_q(wiki, "--ask", "alpha", "Какой рост у Alpha?", "--category", "clarify")
    assert r.returncode == 0, r.stderr
    # Verify file now contains the question
    import yaml
    text = (wiki / "npcs" / "alpha.md").read_text()
    body_idx = text.find("\n---", 3)
    fm = yaml.safe_load(text[3:body_idx])
    assert fm["open_questions"][0]["text"] == "Какой рост у Alpha?"
    assert fm["open_questions"][0]["category"] == "clarify"
    assert fm["open_questions"][0]["answer"] is None
```

- [ ] **Step 2: Run tests, expect failures (script does not exist)**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest tests/test_dnd_questions.py -v
```

Expected: all FAIL (FileNotFoundError).

- [ ] **Step 3: Implement `scripts/dnd-questions.py`**

Create the script:

```python
#!/usr/bin/env python3
"""Open-questions CLI — ask, list, answer, stale.

Data sources:
  - Entity frontmatter `open_questions:` lists (npcs/, quests/, locations/,
    sessions/, characters/, items/)
  - Wiki-wide `questions.md` YAML-fenced entries

Entry shape (entity-scoped):
  - text: str
    category: clarify | lore | blocker
    asked: ISO-date
    asked_in: session-slug (optional)
    answer: str | null
    answered_in: session-slug (optional)

Entry shape (wiki-wide, questions.md):
  - id: q-<tag>-NNN
    text: str
    category: ...
    asked: ISO-date
    answer: str | null
"""
from __future__ import annotations
import argparse
import datetime as dt
import re
import sys
from pathlib import Path

SCRIPT_DIR = Path(__file__).resolve().parent
sys.path.insert(0, str(SCRIPT_DIR.parent))
from lib.kb_common import (  # noqa: E402
    resolve_config, read_frontmatter, write_frontmatter, append_log,
)

import yaml

ENTITY_FOLDERS = ["npcs", "quests", "locations", "sessions", "characters", "items"]


def load_entity_questions(dnd_dir: Path) -> list[dict]:
    """Return list of {entity_slug, entity_path, index, question}."""
    out = []
    for folder in ENTITY_FOLDERS:
        d = dnd_dir / folder
        if not d.exists():
            continue
        for p in sorted(d.glob("*.md")):
            if p.name.startswith("_"):
                continue
            meta, _ = read_frontmatter(p)
            if not meta:
                continue
            qs = meta.get("open_questions") or []
            if not isinstance(qs, list):
                continue
            slug = meta.get("slug", p.stem)
            for i, q in enumerate(qs):
                if not isinstance(q, dict):
                    continue
                if q.get("answer") is not None:
                    continue
                out.append({
                    "entity_slug": slug, "entity_path": p,
                    "index": i, "question": q,
                })
    return out


def load_wiki_questions(dnd_dir: Path) -> list[dict]:
    """Return entries from questions.md (YAML-fenced)."""
    p = dnd_dir / "questions.md"
    if not p.exists():
        return []
    text = p.read_text(encoding="utf-8")
    out = []
    for m in re.finditer(r"```yaml\n(.*?)```", text, re.S):
        try:
            data = yaml.safe_load(m.group(1))
        except yaml.YAMLError:
            continue
        if isinstance(data, list):
            for item in data:
                if isinstance(item, dict) and item.get("answer") is None:
                    out.append({"wiki_id": item.get("id"), "question": item})
    return out


def cmd_list(args, dnd_dir: Path) -> int:
    entity_qs = load_entity_questions(dnd_dir)
    wiki_qs = load_wiki_questions(dnd_dir)
    if args.by_category:
        entity_qs = [x for x in entity_qs if x["question"].get("category") == args.by_category]
        wiki_qs = [x for x in wiki_qs if x["question"].get("category") == args.by_category]
    if args.by_entity:
        entity_qs = [x for x in entity_qs if x["entity_slug"] == args.by_entity]
        wiki_qs = [] if args.by_entity else wiki_qs
    if args.stale is not None:
        cutoff = dt.date.today() - dt.timedelta(days=args.stale)
        def _stale(q):
            asked = q.get("asked")
            if not asked:
                return False
            try:
                return dt.date.fromisoformat(str(asked)) < cutoff
            except ValueError:
                return False
        entity_qs = [x for x in entity_qs if _stale(x["question"])]
        wiki_qs = [x for x in wiki_qs if _stale(x["question"])]
    total = len(entity_qs) + len(wiki_qs)
    if total == 0:
        print("Нет открытых вопросов (по заданному фильтру).")
        return 0
    print(f"=== Открытых вопросов: {total} ===\n")
    if wiki_qs:
        print("── Wiki-wide (questions.md) ──")
        for item in wiki_qs:
            q = item["question"]
            print(f"  [{item['wiki_id']}] ({q.get('category','?')}) {q.get('text','?')}")
            if q.get("asked"):
                print(f"      asked: {q['asked']}")
        print()
    if entity_qs:
        print("── Entity-scoped ──")
        for item in entity_qs:
            q = item["question"]
            print(f"  [{item['entity_slug']}#{item['index']}] "
                  f"({q.get('category','?')}) {q.get('text','?')}")
            if q.get("asked"):
                print(f"      asked: {q['asked']}")
    return 0


def cmd_ask(args, dnd_dir: Path) -> int:
    target_slug = args.entity
    # Locate entity file
    target_path = None
    for folder in ENTITY_FOLDERS:
        p = dnd_dir / folder / f"{target_slug}.md"
        if p.exists():
            target_path = p
            break
    if target_path is None:
        # Wiki-wide — require --id
        if not args.id:
            sys.stderr.write(
                f"entity '{target_slug}' not found; for wiki-wide question use "
                "slug=questions and pass --id q-<tag>-NNN\n"
            )
            return 1
        # Add to questions.md
        qmd = dnd_dir / "questions.md"
        qmd.touch()
        text = qmd.read_text(encoding="utf-8")
        new_entry = (
            f"- id: {args.id}\n"
            f"  text: {args.text!r}\n"
            f"  category: {args.category}\n"
            f"  asked: {dt.date.today().isoformat()}\n"
            f"  answer: null\n"
        )
        if "```yaml" in text:
            text = text.replace("```yaml\n", f"```yaml\n{new_entry}", 1)
        else:
            text = text.rstrip() + f"\n\n```yaml\n{new_entry}```\n"
        qmd.write_text(text, encoding="utf-8")
        print(f"Added wiki-wide question {args.id}")
        append_log(dnd_dir, "note", f"question asked: {args.id}")
        return 0
    # Entity-scoped add
    meta, body = read_frontmatter(target_path)
    qs = meta.get("open_questions") or []
    if not isinstance(qs, list):
        qs = []
    qs.append({
        "text": args.text,
        "category": args.category,
        "asked": dt.date.today().isoformat(),
        "answer": None,
    })
    meta["open_questions"] = qs
    meta["updated"] = dt.date.today().isoformat()
    write_frontmatter(target_path, meta, body)
    print(f"Added question to {target_slug} (index {len(qs)-1})")
    append_log(dnd_dir, "note", f"question asked on {target_slug}")
    return 0


def main():
    parser = argparse.ArgumentParser(description="Open-questions CLI")
    parser.add_argument("--by-category", choices=["clarify", "lore", "blocker"])
    parser.add_argument("--by-entity", metavar="SLUG")
    parser.add_argument("--stale", type=int, metavar="DAYS", nargs="?", const=30)
    parser.add_argument("--ask", nargs=2, metavar=("ENTITY_SLUG", "TEXT"))
    parser.add_argument("--category", default="clarify",
                        choices=["clarify", "lore", "blocker"])
    parser.add_argument("--id", help="ID for wiki-wide questions (q-<tag>-NNN)")
    args = parser.parse_args()

    cfg = resolve_config()
    dnd_dir = cfg["dnd_dir"]

    if args.ask:
        args.entity, args.text = args.ask
        return sys.exit(cmd_ask(args, dnd_dir))
    return sys.exit(cmd_list(args, dnd_dir))


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Make executable**

```bash
chmod +x /Users/claw/.openclaw/skills/dnd/scripts/dnd-questions.py
```

- [ ] **Step 5: Run tests, expect pass**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest tests/test_dnd_questions.py -v
```

Expected: all 5 tests PASS.

- [ ] **Step 6: Sanity-check CLI against real wiki**

```bash
python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-questions.py
```

Expected: lists open_questions from Phase 3 (main-luskan, 4 characters hp_max, 2 characters subclass) + 3 wiki-wide from questions.md. Total ≈ 10.

- [ ] **Step 7: Commit**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/scripts/dnd-questions.py skills/dnd/tests/test_dnd_questions.py
git commit -m "$(cat <<'EOF'
feat: dnd-questions.py CLI — list and ask commands

Supports:
  dnd-questions.py                          # list all open
  dnd-questions.py --by-category blocker    # filter
  dnd-questions.py --by-entity <slug>       # scope
  dnd-questions.py --stale [days=30]        # stale only
  dnd-questions.py --ask <slug> "text" [--category ...]

--answer comes in Task 14.

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.7

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 14: Implement `dnd-questions.py --answer` + tests

**Files:**
- Modify: `/Users/claw/.openclaw/skills/dnd/scripts/dnd-questions.py`
- Modify: `/Users/claw/.openclaw/skills/dnd/tests/test_dnd_questions.py`

- [ ] **Step 1: Write failing test for --answer command**

Append to `tests/test_dnd_questions.py`:

```python
def test_answer_removes_entry_and_adds_to_body(wiki):
    (wiki / "npcs" / "alpha.md").write_text(textwrap.dedent("""\
        ---
        name: Alpha
        slug: alpha
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: нейтральный
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        open_questions:
          - text: "Какой рост?"
            category: clarify
            asked: 2026-04-24
            answer: null
        ---

        body text
    """))
    r = run_q(wiki, "--answer", "alpha", "0", "180 см")
    assert r.returncode == 0, r.stderr
    import yaml
    text = (wiki / "npcs" / "alpha.md").read_text()
    body_idx = text.find("\n---", 3)
    fm = yaml.safe_load(text[3:body_idx])
    body = text[body_idx + 4:]
    # Frontmatter entry removed
    assert fm.get("open_questions", []) == []
    # Answer moved to body
    assert "180 см" in body
    assert "Какой рост?" in body  # question preserved as context


def test_answer_wiki_wide_marks_closed(wiki):
    (wiki / "questions.md").write_text(textwrap.dedent("""\
        # Questions

        ```yaml
        - id: q-tst-001
          text: "Global?"
          category: lore
          asked: 2026-04-24
          answer: null
        ```
    """))
    r = run_q(wiki, "--answer", "q-tst-001", "Yes")
    assert r.returncode == 0, r.stderr
    text = (wiki / "questions.md").read_text()
    assert "answer: Yes" in text or "answer: 'Yes'" in text
```

- [ ] **Step 2: Run tests, expect failure**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest tests/test_dnd_questions.py::test_answer_removes_entry_and_adds_to_body tests/test_dnd_questions.py::test_answer_wiki_wide_marks_closed -v
```

Expected: FAIL (--answer not implemented).

- [ ] **Step 3: Implement cmd_answer in `dnd-questions.py`**

Add the function before `def main()`:

```python
def cmd_answer(args, dnd_dir: Path) -> int:
    # Detect wiki-wide vs entity-scoped by slug pattern
    slug_or_id = args.entity_or_id
    if re.match(r"^q-[a-z]{3,5}-\d{3}$", slug_or_id):
        # Wiki-wide
        qmd = dnd_dir / "questions.md"
        if not qmd.exists():
            sys.stderr.write("questions.md does not exist\n")
            return 1
        text = qmd.read_text(encoding="utf-8")
        # Parse YAML, find by id, set answer
        def _replace_block(match):
            data = yaml.safe_load(match.group(1))
            if not isinstance(data, list):
                return match.group(0)
            changed = False
            for entry in data:
                if isinstance(entry, dict) and entry.get("id") == slug_or_id:
                    entry["answer"] = args.answer
                    entry["answered"] = dt.date.today().isoformat()
                    if args.answered_in:
                        entry["answered_in"] = args.answered_in
                    changed = True
            if not changed:
                return match.group(0)
            return "```yaml\n" + yaml.safe_dump(
                data, allow_unicode=True, sort_keys=False
            ) + "```"
        new_text = re.sub(r"```yaml\n(.*?)```", _replace_block, text, flags=re.S)
        qmd.write_text(new_text, encoding="utf-8")
        print(f"Answered {slug_or_id}")
        append_log(dnd_dir, "note", f"question answered: {slug_or_id}")
        return 0
    # Entity-scoped
    for folder in ENTITY_FOLDERS:
        p = dnd_dir / folder / f"{slug_or_id}.md"
        if p.exists():
            break
    else:
        sys.stderr.write(f"entity '{slug_or_id}' not found\n")
        return 1
    meta, body = read_frontmatter(p)
    qs = meta.get("open_questions") or []
    try:
        idx = int(args.index)
    except (ValueError, TypeError):
        sys.stderr.write("--answer requires integer index for entity-scoped\n")
        return 1
    if idx < 0 or idx >= len(qs):
        sys.stderr.write(f"index {idx} out of range (0..{len(qs)-1})\n")
        return 1
    question = qs.pop(idx)
    # Append to body under "## Известные факты" (create section if missing)
    qtext = question.get("text", "?")
    answer_line = f"- **{qtext}** — {args.answer} _(подтверждено {dt.date.today().isoformat()})_\n"
    if "## Известные факты" in body:
        body = body.replace("## Известные факты\n", "## Известные факты\n" + answer_line, 1)
    else:
        body = body.rstrip() + "\n\n## Известные факты\n\n" + answer_line
    meta["open_questions"] = qs
    if not qs:
        del meta["open_questions"]
    meta["updated"] = dt.date.today().isoformat()
    write_frontmatter(p, meta, body)
    print(f"Answered {slug_or_id}#{idx}: {qtext}")
    append_log(dnd_dir, "note", f"question answered: {slug_or_id}#{idx}")
    return 0
```

Add to argparse in `main()`:

```python
    parser.add_argument("--answer", nargs="+",
                        metavar="(SLUG_OR_ID INDEX ANSWER | ID ANSWER)",
                        help="Answer a question. Entity: slug index answer. "
                             "Wiki-wide: id answer.")
    parser.add_argument("--answered-in", metavar="SESSION_SLUG")
```

Replace the `if args.ask:` branch with dispatching that also handles `--answer`:

```python
    if args.answer:
        if len(args.answer) == 3:
            args.entity_or_id, args.index, args.answer = args.answer
        elif len(args.answer) == 2:
            args.entity_or_id, args.answer = args.answer
            args.index = None
        else:
            sys.stderr.write("--answer expects 2 args (wiki-id answer) or 3 "
                             "args (entity-slug index answer)\n")
            return sys.exit(1)
        return sys.exit(cmd_answer(args, dnd_dir))
    if args.ask:
        args.entity, args.text = args.ask
        return sys.exit(cmd_ask(args, dnd_dir))
    return sys.exit(cmd_list(args, dnd_dir))
```

- [ ] **Step 4: Run tests, expect pass**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest tests/test_dnd_questions.py -v
```

Expected: all tests PASS.

- [ ] **Step 5: Commit**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/scripts/dnd-questions.py skills/dnd/tests/test_dnd_questions.py
git commit -m "$(cat <<'EOF'
feat: dnd-questions.py --answer + --answered-in

Closes an open question — entity-scoped (slug + index + answer) or
wiki-wide (id + answer). Entity entries move answer to body section
## Известные факты and delete frontmatter entry. Wiki-wide entries
are marked answer: <text>, answered: <date>.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 15: Lint check L7 — `unresolved-placeholders`

**Files:**
- Modify: `/Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py`
- Create: `/Users/claw/.openclaw/skills/dnd/tests/test_dnd_lint_placeholders.py`

- [ ] **Step 1: Write failing tests for placeholder detection**

Create `tests/test_dnd_lint_placeholders.py`:

```python
"""Tests for unresolved-placeholders lint check (L7)."""
from __future__ import annotations
import json
import os
import subprocess
import sys
import textwrap
from pathlib import Path

SKILL_ROOT = Path(__file__).resolve().parent.parent
LINT = SKILL_ROOT / "scripts" / "dnd-lint.py"


def run_lint(wiki_dir: Path, *args: str) -> dict:
    env = {**os.environ, "DND_DIR": str(wiki_dir)}
    r = subprocess.run(
        [sys.executable, str(LINT), *args, "--format", "json"],
        capture_output=True, text=True, env=env,
    )
    return json.loads(r.stdout)


def test_placeholder_tbd_detected(wiki):
    (wiki / "characters" / "test.md").write_text(textwrap.dedent("""\
        ---
        name: T
        slug: test
        type: character
        player: P
        race: x
        classes: [маг]
        level: 1
        status: healthy
        subclass: TBD
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        ---

        body
    """))
    data = run_lint(wiki, "--only", "unresolved-placeholders")
    assert any(
        i["check"] == "unresolved-placeholders"
        and "subclass" in (i.get("field") or "")
        and "TBD" in i["message"]
        for i in data["issues"]
    ), data


def test_placeholder_utochnit_detected(wiki):
    (wiki / "characters" / "test.md").write_text(textwrap.dedent("""\
        ---
        name: T
        slug: test
        type: character
        player: P
        race: x
        classes: [маг]
        level: 1
        status: healthy
        subclass: уточнить
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        ---

        body
    """))
    data = run_lint(wiki, "--only", "unresolved-placeholders")
    assert any(
        i["check"] == "unresolved-placeholders"
        and "уточнить" in i["message"].lower()
        for i in data["issues"]
    ), data


def test_placeholder_in_enum_field_ignored(wiki):
    """status=неизвестно is a legitimate enum value — no warning."""
    (wiki / "npcs" / "test.md").write_text(textwrap.dedent("""\
        ---
        name: T
        slug: test
        type: npc
        race: x
        role: x
        location: test-loc
        status: неизвестно
        relation: неизвестно
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        ---

        body
    """))
    data = run_lint(wiki, "--only", "unresolved-placeholders")
    # неизвестно is in npc.status enum → not a placeholder
    assert not any(
        i["check"] == "unresolved-placeholders" and i["field"] in ("status", "relation")
        for i in data["issues"]
    ), data


def test_placeholder_clean_wiki_no_issues(wiki):
    data = run_lint(wiki, "--only", "unresolved-placeholders")
    assert not any(i["check"] == "unresolved-placeholders" for i in data["issues"])
```

- [ ] **Step 2: Run tests, expect failures**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest tests/test_dnd_lint_placeholders.py -v
```

Expected: all FAIL (no such check registered).

- [ ] **Step 3: Implement L7 check in `dnd-lint.py`**

Add after the existing `@register("stale-focus")` block:

```python
PLACEHOLDER_PATTERNS = [
    r"^TBD$", r"^TODO$", r"^FIXME$",
    r"^уточнить$", r"^не выбрана$", r"^не выбран$",
    r"^\?$", r"^\?\?\?$",
    r"^неизвестно$",  # only flagged when field enum does NOT allow it
]

_PLACEHOLDER_RE = re.compile("|".join(PLACEHOLDER_PATTERNS), re.IGNORECASE)


def _field_allows_placeholder(schema: dict, type_name: str, field: str, value: str) -> bool:
    """True if the value is in the enum for (type_name, field) — legitimate, not a placeholder."""
    enums = schema.get("enums", {}).get(type_name, {})
    allowed = enums.get(field)
    if not allowed:
        return False
    return value in allowed


@register("unresolved-placeholders")
def check_unresolved_placeholders(*, dnd_dir, schema, pages) -> list[Issue]:
    out = []
    for p in pages:
        meta = p["meta"]
        type_name = meta.get("type")
        for field, value in meta.items():
            if field in ("open_questions", "tags", "sources", "mentions", "relations",
                         "related_npcs", "related_locations"):
                continue
            if isinstance(value, str) and _PLACEHOLDER_RE.match(value.strip()):
                if _field_allows_placeholder(schema, type_name, field, value.strip()):
                    continue
                out.append(Issue(
                    check="unresolved-placeholders", severity="warning",
                    file=p["path"], field=field,
                    message=f"placeholder value '{value}' — consider moving to open_questions",
                    fixable=False,
                ))
    return out
```

- [ ] **Step 4: Run tests, expect pass**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest tests/test_dnd_lint_placeholders.py -v
```

Expected: all 4 tests PASS.

- [ ] **Step 5: Run full test suite + real wiki lint to confirm**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py
```

Expected: all skill tests pass; real wiki lint has errors=0, warnings=0 (Phase 3 already migrated placeholders to `null + open_questions`).

- [ ] **Step 6: Commit**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/scripts/dnd-lint.py skills/dnd/tests/test_dnd_lint_placeholders.py
git commit -m "$(cat <<'EOF'
feat(lint): unresolved-placeholders check (L7)

Detects placeholder tokens in frontmatter values: TBD, TODO, FIXME,
уточнить, не выбрана, ?, ???, неизвестно. Respects enum membership
(неизвестно is a valid npc.status value, not a placeholder there).

Severity: warning (error in --strict). Suggests moving to open_questions.

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.3 L7

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 16: Lint check L8 — `stale-open-question`

**Files:**
- Modify: `/Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py`
- Create: `/Users/claw/.openclaw/skills/dnd/tests/test_dnd_lint_stale_questions.py`

- [ ] **Step 1: Write failing tests**

Create `tests/test_dnd_lint_stale_questions.py`:

```python
"""Tests for stale-open-question lint check (L8)."""
from __future__ import annotations
import json
import os
import subprocess
import sys
import textwrap
from pathlib import Path

SKILL_ROOT = Path(__file__).resolve().parent.parent
LINT = SKILL_ROOT / "scripts" / "dnd-lint.py"


def run_lint(wiki_dir: Path, *args: str) -> dict:
    env = {**os.environ, "DND_DIR": str(wiki_dir)}
    r = subprocess.run(
        [sys.executable, str(LINT), *args, "--format", "json"],
        capture_output=True, text=True, env=env,
    )
    return json.loads(r.stdout)


def test_stale_question_flagged(wiki):
    (wiki / "npcs" / "alpha.md").write_text(textwrap.dedent("""\
        ---
        name: A
        slug: alpha
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: нейтральный
        tags: []
        sources: []
        created: 2020-01-01
        updated: 2020-01-01
        open_questions:
          - text: "Old question?"
            category: clarify
            asked: 2020-01-01
            answer: null
        ---

        body
    """))
    data = run_lint(wiki, "--only", "stale-open-question")
    assert any(
        i["check"] == "stale-open-question" and i["severity"] == "info"
        and "Old question?" in i["message"]
        for i in data["issues"]
    ), data


def test_recent_question_not_flagged(wiki):
    import datetime as dt
    today = dt.date.today().isoformat()
    (wiki / "npcs" / "alpha.md").write_text(textwrap.dedent(f"""\
        ---
        name: A
        slug: alpha
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: нейтральный
        tags: []
        sources: []
        created: {today}
        updated: {today}
        open_questions:
          - text: "Fresh?"
            category: clarify
            asked: {today}
            answer: null
        ---

        body
    """))
    data = run_lint(wiki, "--only", "stale-open-question")
    assert not any(i["check"] == "stale-open-question" for i in data["issues"])
```

- [ ] **Step 2: Run tests, expect failure**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest tests/test_dnd_lint_stale_questions.py -v
```

Expected: all FAIL.

- [ ] **Step 3: Implement check**

Add to `dnd-lint.py`:

```python
STALE_OPEN_QUESTION_DAYS = 30


@register("stale-open-question")
def check_stale_open_question(*, dnd_dir, schema, pages) -> list[Issue]:
    out = []
    today = date.today()
    for p in pages:
        meta = p["meta"]
        qs = meta.get("open_questions") or []
        if not isinstance(qs, list):
            continue
        for i, q in enumerate(qs):
            if not isinstance(q, dict):
                continue
            if q.get("answer") is not None:
                continue
            asked = q.get("asked")
            if not asked:
                continue
            try:
                asked_date = date.fromisoformat(str(asked))
            except ValueError:
                continue
            if (today - asked_date).days > STALE_OPEN_QUESTION_DAYS:
                out.append(Issue(
                    check="stale-open-question", severity="info",
                    file=p["path"], field=f"open_questions[{i}]",
                    message=f"{q.get('text','?')} (asked {asked}, >{STALE_OPEN_QUESTION_DAYS}d ago)",
                    fixable=False,
                ))
    # Also scan questions.md
    qmd = dnd_dir / "questions.md"
    if qmd.exists():
        text = qmd.read_text(encoding="utf-8")
        for m in re.finditer(r"```yaml\n(.*?)```", text, re.S):
            try:
                data = _yaml.safe_load(m.group(1))
            except _yaml.YAMLError:
                continue
            if not isinstance(data, list):
                continue
            for entry in data:
                if not isinstance(entry, dict) or entry.get("answer") is not None:
                    continue
                asked = entry.get("asked")
                if not asked:
                    continue
                try:
                    asked_date = date.fromisoformat(str(asked))
                except ValueError:
                    continue
                if (today - asked_date).days > STALE_OPEN_QUESTION_DAYS:
                    out.append(Issue(
                        check="stale-open-question", severity="info",
                        file=qmd, field=entry.get("id", "?"),
                        message=f"{entry.get('text','?')} (asked {asked})",
                        fixable=False,
                    ))
    return out
```

- [ ] **Step 4: Run tests, expect pass**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest tests/test_dnd_lint_stale_questions.py -v
```

Expected: both tests PASS.

- [ ] **Step 5: Run full suite + real wiki**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py
```

Expected: all green; real wiki — questions created today are not stale yet.

- [ ] **Step 6: Commit**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/scripts/dnd-lint.py skills/dnd/tests/test_dnd_lint_stale_questions.py
git commit -m "$(cat <<'EOF'
feat(lint): stale-open-question check (L8)

Info-severity flag for open_questions with asked > 30 days ago and
answer=null. Scans entity frontmatter AND questions.md YAML entries.

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.3 L8

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Phase 6: Seven additional lint checks

Phase goal: land L9, L2, L3, L1, L4, L5, L6 — in that implementation order. After this phase, `dnd-lint.py` has **24 checks** total (15 existing + 9 new).

### Task 17: Lint check L9 — `schema-undeclared-fields`

**Files:**
- Modify: `/Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py`
- Create: `/Users/claw/.openclaw/skills/dnd/tests/test_dnd_lint_undeclared_fields.py`

- [ ] **Step 1: Write failing tests**

Create `tests/test_dnd_lint_undeclared_fields.py`:

```python
"""Tests for schema-undeclared-fields lint check (L9)."""
from __future__ import annotations
import json
import os
import subprocess
import sys
import textwrap
from pathlib import Path

SKILL_ROOT = Path(__file__).resolve().parent.parent
LINT = SKILL_ROOT / "scripts" / "dnd-lint.py"


def run_lint(wiki_dir: Path, *args: str) -> dict:
    env = {**os.environ, "DND_DIR": str(wiki_dir)}
    r = subprocess.run(
        [sys.executable, str(LINT), *args, "--format", "json"],
        capture_output=True, text=True, env=env,
    )
    return json.loads(r.stdout)


def test_undeclared_field_flagged(wiki):
    # Add optional blocks to schema
    schema = wiki / "SCHEMA.md"
    schema.write_text(schema.read_text() + textwrap.dedent("""

        <!-- schema:optional:npc -->
        - first_seen
        <!-- /schema:optional:npc -->

        <!-- schema:optional:common -->
        - aliases
        <!-- /schema:optional:common -->
    """))
    (wiki / "npcs" / "alpha.md").write_text(textwrap.dedent("""\
        ---
        name: A
        slug: alpha
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: нейтральный
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        some_weird_field: something
        ---

        body
    """))
    data = run_lint(wiki, "--only", "schema-undeclared-fields")
    assert any(
        i["check"] == "schema-undeclared-fields"
        and "some_weird_field" in i["message"]
        for i in data["issues"]
    ), data


def test_declared_optional_not_flagged(wiki):
    schema = wiki / "SCHEMA.md"
    schema.write_text(schema.read_text() + textwrap.dedent("""

        <!-- schema:optional:npc -->
        - first_seen
        <!-- /schema:optional:npc -->
    """))
    (wiki / "npcs" / "alpha.md").write_text(textwrap.dedent("""\
        ---
        name: A
        slug: alpha
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: нейтральный
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        first_seen: 2026-01-01
        ---

        body
    """))
    data = run_lint(wiki, "--only", "schema-undeclared-fields")
    assert not any(
        i["check"] == "schema-undeclared-fields" and i["field"] == "first_seen"
        for i in data["issues"]
    )


def test_required_field_not_flagged(wiki):
    (wiki / "npcs" / "alpha.md").write_text(textwrap.dedent("""\
        ---
        name: A
        slug: alpha
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: нейтральный
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        ---

        body
    """))
    data = run_lint(wiki, "--only", "schema-undeclared-fields")
    assert not any(i["check"] == "schema-undeclared-fields" for i in data["issues"])
```

- [ ] **Step 2: Run, expect failure**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest tests/test_dnd_lint_undeclared_fields.py -v
```

Expected: all 3 FAIL.

- [ ] **Step 3: Implement check**

Add to `dnd-lint.py`:

```python
# Fields always allowed regardless of type (implicit frontmatter structure)
ALWAYS_ALLOWED_FIELDS = {
    "name", "slug", "type", "created", "updated", "tags", "sources",
    "mentions", "relations", "contradictions",
}


@register("schema-undeclared-fields")
def check_schema_undeclared_fields(*, dnd_dir, schema, pages) -> list[Issue]:
    required = schema.get("required", {})
    optional = schema.get("optional", {})
    enums = schema.get("enums", {})
    out = []
    for p in pages:
        meta = p["meta"]
        type_name = meta.get("type")
        if not type_name:
            continue
        allowed = set(ALWAYS_ALLOWED_FIELDS)
        allowed.update(required.get(type_name, []))
        allowed.update(optional.get(type_name, []))
        allowed.update(optional.get("common", []))
        allowed.update(enums.get(type_name, {}).keys())
        for field in meta.keys():
            if field not in allowed:
                out.append(Issue(
                    check="schema-undeclared-fields", severity="warning",
                    file=p["path"], field=field,
                    message=f"field '{field}' not declared in SCHEMA required/optional "
                            f"for type '{type_name}' — add to SCHEMA or remove",
                    fixable=False,
                ))
    return out
```

- [ ] **Step 4: Run tests, expect pass**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest tests/test_dnd_lint_undeclared_fields.py -v
```

- [ ] **Step 5: Run real-wiki lint to see if anything slipped through**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py --only schema-undeclared-fields
```

Expected: 0 warnings (Phase 1 documented everything; Phase 3 removed `secrets:`). If any pop up, add the field to SCHEMA optional block and re-run.

- [ ] **Step 6: Full regression**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest
```

- [ ] **Step 7: Commit**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/scripts/dnd-lint.py skills/dnd/tests/test_dnd_lint_undeclared_fields.py
git commit -m "$(cat <<'EOF'
feat(lint): schema-undeclared-fields check (L9)

Flags frontmatter fields that are not declared as required/optional
in SCHEMA for the page type. Catches invented fields early.

Severity: warning. Auto-fix: not safe (may contain real data).

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.3 L9

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 18: Lint check L2 — `body-slug-mentions` (with --fix support)

**Files:**
- Modify: `/Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py`
- Create: `/Users/claw/.openclaw/skills/dnd/tests/test_dnd_lint_body_mentions.py`

- [ ] **Step 1: Write failing tests**

Create `tests/test_dnd_lint_body_mentions.py`:

```python
"""Tests for body-slug-mentions lint check (L2)."""
from __future__ import annotations
import json
import os
import subprocess
import sys
import textwrap
from pathlib import Path

SKILL_ROOT = Path(__file__).resolve().parent.parent
LINT = SKILL_ROOT / "scripts" / "dnd-lint.py"


def run_lint(wiki_dir: Path, *args: str) -> dict:
    env = {**os.environ, "DND_DIR": str(wiki_dir)}
    r = subprocess.run(
        [sys.executable, str(LINT), *args, "--format", "json"],
        capture_output=True, text=True, env=env,
    )
    return json.loads(r.stdout)


def test_body_markdown_link_missing_from_relations_flagged(wiki):
    # alpha referenced in body via markdown link but not in frontmatter
    (wiki / "npcs" / "alpha.md").write_text(textwrap.dedent("""\
        ---
        name: A
        slug: alpha
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: нейтральный
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        ---

        body
    """))
    (wiki / "npcs" / "beta.md").write_text(textwrap.dedent("""\
        ---
        name: B
        slug: beta
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: союзник
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        ---

        Beta сотрудничает с [Alpha](../npcs/alpha.md) — детали тут.
    """))
    data = run_lint(wiki, "--only", "body-slug-mentions")
    assert any(
        i["check"] == "body-slug-mentions" and "alpha" in i["message"]
        and "beta.md" in str(i["file"])
        for i in data["issues"]
    ), data


def test_body_relation_present_no_warning(wiki):
    (wiki / "npcs" / "alpha.md").write_text(textwrap.dedent("""\
        ---
        name: A
        slug: alpha
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: нейтральный
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        ---

        body
    """))
    (wiki / "npcs" / "beta.md").write_text(textwrap.dedent("""\
        ---
        name: B
        slug: beta
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: союзник
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        relations:
          - to: alpha
            kind: союзник
        ---

        Beta сотрудничает с [Alpha](../npcs/alpha.md).
    """))
    data = run_lint(wiki, "--only", "body-slug-mentions")
    assert not any(
        i["check"] == "body-slug-mentions" and i["file"].endswith("beta.md")
        for i in data["issues"]
    )


def test_fix_appends_slug_to_mentions(wiki):
    (wiki / "npcs" / "alpha.md").write_text(textwrap.dedent("""\
        ---
        name: A
        slug: alpha
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: нейтральный
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        ---

        body
    """))
    (wiki / "npcs" / "beta.md").write_text(textwrap.dedent("""\
        ---
        name: B
        slug: beta
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: союзник
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        ---

        Beta сотрудничает с [Alpha](../npcs/alpha.md).
    """))
    env = {**os.environ, "DND_DIR": str(wiki)}
    subprocess.run(
        [sys.executable, str(LINT), "--fix", "--only", "body-slug-mentions"],
        capture_output=True, env=env,
    )
    import yaml
    text = (wiki / "npcs" / "beta.md").read_text()
    body_idx = text.find("\n---", 3)
    fm = yaml.safe_load(text[3:body_idx])
    assert "alpha" in (fm.get("mentions") or [])
```

- [ ] **Step 2: Run, expect failure**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest tests/test_dnd_lint_body_mentions.py -v
```

Expected: all FAIL.

- [ ] **Step 3: Implement check + fixer**

Add to `dnd-lint.py`:

```python
_MD_LINK_RE = re.compile(r"\[[^\]]+\]\(\.\.\/[^\/]+\/([a-z0-9-]+)\.md\)")


@register("body-slug-mentions")
def check_body_slug_mentions(*, dnd_dir, schema, pages) -> list[Issue]:
    slugs = all_slugs(pages)
    out = []
    for p in pages:
        body = p["body"] or ""
        meta = p["meta"]
        own_slug = meta.get("slug", "")
        referenced = set(m.group(1) for m in _MD_LINK_RE.finditer(body))
        referenced.discard(own_slug)
        referenced = {s for s in referenced if s in slugs}
        known_refs = set()
        for _fld, slug in _iter_refs(meta, p["type"]):
            known_refs.add(slug)
        mentions = meta.get("mentions") or []
        if isinstance(mentions, list):
            known_refs.update(s for s in mentions if isinstance(s, str))
        for slug in sorted(referenced - known_refs):
            out.append(Issue(
                check="body-slug-mentions", severity="warning",
                file=p["path"], field="mentions",
                message=f"body links to '{slug}' but slug not in relations/mentions "
                        f"(run --fix to auto-append)",
                fixable=True,
            ))
    return out
```

Then extend `apply_fixes` to handle `body-slug-mentions`:

```python
def apply_fixes(dnd_dir: Path, issues: list[Issue]) -> int:
    from lib.kb_common import read_frontmatter, write_frontmatter
    fixed = 0
    had_missing_index = any(i.check == "missing-index" for i in issues if i.fixable)
    had_drift = any(i.check in ("state-drift", "campaign-drift", "stale-focus")
                    for i in issues if i.fixable)
    # body-slug-mentions autofix
    by_file: dict[Path, list[str]] = {}
    for i in issues:
        if i.check == "body-slug-mentions" and i.fixable:
            slug = i.message.split("'")[1]
            by_file.setdefault(i.file, []).append(slug)
    for path, slugs in by_file.items():
        meta, body = read_frontmatter(path)
        mentions = list(meta.get("mentions") or [])
        for s in slugs:
            if s not in mentions:
                mentions.append(s)
        if mentions != (meta.get("mentions") or []):
            meta["mentions"] = mentions
            meta["updated"] = date.today().isoformat()
            write_frontmatter(path, meta, body)
            fixed += len(slugs)
    # Existing autofix for missing-required updated
    for i in issues:
        if i.check == "missing-required" and i.field == "updated":
            meta, body = read_frontmatter(i.file)
            meta["updated"] = date.today().isoformat()
            write_frontmatter(i.file, meta, body)
            fixed += 1
    # Existing drift autofix
    if had_drift:
        import subprocess
        import os as _os
        subprocess.run(
            [sys.executable, str(SCRIPT_DIR / "dnd-sync-state.py"), "--fix"],
            env={**_os.environ, "DND_DIR": str(dnd_dir)},
            check=False,
        )
        fixed += sum(1 for i in issues if i.check in (
            "state-drift", "campaign-drift", "stale-focus") and i.fixable)
    if had_missing_index:
        import subprocess
        import os as _os
        subprocess.run(
            [sys.executable, str(SCRIPT_DIR / "dnd-rebuild.py")],
            env={**_os.environ, "DND_DIR": str(dnd_dir)},
            check=False,
        )
        fixed += sum(1 for i in issues if i.check == "missing-index")
    return fixed
```

- [ ] **Step 4: Run tests, expect pass**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest tests/test_dnd_lint_body_mentions.py -v
```

- [ ] **Step 5: Dry-run on real wiki to see how many warnings pop up**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py --only body-slug-mentions
```

Expected: multiple warnings on entity files that link in body but lack frontmatter refs. Real number will be discovered during execution.

- [ ] **Step 6: Apply --fix on real wiki**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py --fix --only body-slug-mentions
```

Expected: auto-fix N entries, re-run shows clean.

- [ ] **Step 7: Commit skill code + data (fixup)**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/scripts/dnd-lint.py skills/dnd/tests/test_dnd_lint_body_mentions.py
git commit -m "$(cat <<'EOF'
feat(lint): body-slug-mentions check (L2) with --fix

Scans body for markdown links ../folder/slug.md; warns if target slug
not referenced in frontmatter relations/mentions. --fix auto-appends
to mentions list and bumps updated.

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.3 L2

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

```bash
cd /Users/claw/PROJECTS/dnd-wiki
git add -A
git commit -m "$(cat <<'EOF'
fix(data): backfill mentions via body-slug-mentions --fix

Auto-applied by dnd-lint.py --fix --only body-slug-mentions.
Every page that linked to another slug in body now has that slug in
its mentions list.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 19: Lint check L3 — `session-mentions-completeness`

**Files:**
- Modify: `/Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py`
- Create: `/Users/claw/.openclaw/skills/dnd/tests/test_dnd_lint_session_mentions.py`

- [ ] **Step 1: Write failing tests**

Create `tests/test_dnd_lint_session_mentions.py`:

```python
"""Tests for session-mentions-completeness lint check (L3)."""
from __future__ import annotations
import json
import os
import subprocess
import sys
import textwrap
from pathlib import Path

SKILL_ROOT = Path(__file__).resolve().parent.parent
LINT = SKILL_ROOT / "scripts" / "dnd-lint.py"


def run_lint(wiki_dir: Path, *args: str) -> dict:
    env = {**os.environ, "DND_DIR": str(wiki_dir)}
    r = subprocess.run(
        [sys.executable, str(LINT), *args, "--format", "json"],
        capture_output=True, text=True, env=env,
    )
    return json.loads(r.stdout)


def test_session_mentions_npc_missing_backlink_error(wiki):
    # session lists alpha in mentions but alpha.mentions does NOT include session
    (wiki / "npcs" / "alpha.md").write_text(textwrap.dedent("""\
        ---
        name: A
        slug: alpha
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: нейтральный
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        ---

        body
    """))
    (wiki / "sessions" / "2026-01-01.md").write_text(textwrap.dedent("""\
        ---
        name: Session
        slug: '2026-01-01'
        type: session
        number: 1
        date: 2026-01-01
        summary_one_line: test
        mentions: [alpha]
        tags: []
        sources: []
        created: 2026-01-01
        updated: 2026-01-01
        ---

        body
    """))
    data = run_lint(wiki, "--only", "session-mentions-completeness")
    assert any(
        i["check"] == "session-mentions-completeness" and i["severity"] == "error"
        and "alpha" in i["message"] and "2026-01-01" in i["message"]
        for i in data["issues"]
    ), data


def test_session_mentions_symmetric_no_error(wiki):
    (wiki / "npcs" / "alpha.md").write_text(textwrap.dedent("""\
        ---
        name: A
        slug: alpha
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: нейтральный
        mentions: ['2026-01-01']
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        ---

        body
    """))
    (wiki / "sessions" / "2026-01-01.md").write_text(textwrap.dedent("""\
        ---
        name: Session
        slug: '2026-01-01'
        type: session
        number: 1
        date: 2026-01-01
        summary_one_line: test
        mentions: [alpha]
        tags: []
        sources: []
        created: 2026-01-01
        updated: 2026-01-01
        ---

        body
    """))
    data = run_lint(wiki, "--only", "session-mentions-completeness")
    assert not any(i["check"] == "session-mentions-completeness" for i in data["issues"])
```

- [ ] **Step 2: Run tests, expect failure**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest tests/test_dnd_lint_session_mentions.py -v
```

- [ ] **Step 3: Implement check**

Add to `dnd-lint.py`:

```python
@register("session-mentions-completeness")
def check_session_mentions_completeness(*, dnd_dir, schema, pages) -> list[Issue]:
    sessions = [p for p in pages if p["type"] == "session"]
    entities = {p["meta"].get("slug"): p for p in pages if p["meta"].get("slug")}
    out = []
    for sess in sessions:
        sess_slug = sess["meta"].get("slug")
        sess_mentions = sess["meta"].get("mentions") or []
        if not isinstance(sess_mentions, list):
            continue
        for entity_slug in sess_mentions:
            if not isinstance(entity_slug, str):
                continue
            target = entities.get(entity_slug)
            if target is None:
                # Already handled by broken-ref; skip
                continue
            target_mentions = target["meta"].get("mentions") or []
            if not isinstance(target_mentions, list):
                target_mentions = []
            if sess_slug not in target_mentions:
                out.append(Issue(
                    check="session-mentions-completeness", severity="error",
                    file=target["path"], field="mentions",
                    message=f"'{entity_slug}' listed in session {sess_slug}.mentions "
                            f"but its own mentions lacks the session slug",
                    fixable=True,
                ))
    return out
```

Extend `apply_fixes` to auto-fill reverse mentions:

```python
    # session-mentions-completeness autofix
    from lib.kb_common import read_frontmatter as _rfm, write_frontmatter as _wfm
    for i in issues:
        if i.check == "session-mentions-completeness" and i.fixable:
            meta, body = _rfm(i.file)
            sess_slug = i.message.split("session ")[1].split(".mentions")[0]
            ms = list(meta.get("mentions") or [])
            if sess_slug not in ms:
                ms.append(sess_slug)
                meta["mentions"] = ms
                meta["updated"] = date.today().isoformat()
                _wfm(i.file, meta, body)
                fixed += 1
```

- [ ] **Step 4: Run tests, expect pass**

- [ ] **Step 5: Apply --fix on real wiki**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py --fix --only session-mentions-completeness
```

Expected: all 33 NPCs (that sessions mention) get session-slugs in their `mentions:`.

- [ ] **Step 6: Verify lint green**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py --strict
```

- [ ] **Step 7: Commit skill + data**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/scripts/dnd-lint.py skills/dnd/tests/test_dnd_lint_session_mentions.py
git commit -m "$(cat <<'EOF'
feat(lint): session-mentions-completeness check (L3)

Error-severity bidirectional check — if session.mentions contains
entity-slug X, entity X.mentions must contain the session-slug.
--fix auto-fills the reverse mention.

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.3 L3

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

```bash
cd /Users/claw/PROJECTS/dnd-wiki
git add -A
git commit -m "fix(data): backfill reverse session mentions on all entities

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 20: Lint check L1 — `bidirectional-relations`

**Files:**
- Modify: `/Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py`
- Create: `/Users/claw/.openclaw/skills/dnd/tests/test_dnd_lint_bidirectional.py`

- [ ] **Step 1: Write failing tests**

Create `tests/test_dnd_lint_bidirectional.py`:

```python
"""Tests for bidirectional-relations lint check (L1)."""
from __future__ import annotations
import json
import os
import subprocess
import sys
import textwrap
from pathlib import Path

SKILL_ROOT = Path(__file__).resolve().parent.parent
LINT = SKILL_ROOT / "scripts" / "dnd-lint.py"


def run_lint(wiki_dir: Path, *args: str) -> dict:
    env = {**os.environ, "DND_DIR": str(wiki_dir)}
    r = subprocess.run(
        [sys.executable, str(LINT), *args, "--format", "json"],
        capture_output=True, text=True, env=env,
    )
    return json.loads(r.stdout)


def _write_npc(wiki, slug, relations=None):
    content = textwrap.dedent(f"""\
        ---
        name: {slug.upper()}
        slug: {slug}
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: нейтральный
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
    """)
    if relations:
        content += "relations:\n"
        for r in relations:
            content += f"  - to: {r['to']}\n    kind: {r['kind']}\n"
    content += "---\n\nbody\n"
    (wiki / "npcs" / f"{slug}.md").write_text(content)


def test_one_way_relation_flagged(wiki):
    _write_npc(wiki, "alpha", relations=[{"to": "beta", "kind": "союзник"}])
    _write_npc(wiki, "beta")  # no relations back
    data = run_lint(wiki, "--only", "bidirectional-relations")
    assert any(
        i["check"] == "bidirectional-relations" and "beta" in i["message"]
        and "alpha" in i["message"]
        for i in data["issues"]
    ), data


def test_mutual_relations_no_warning(wiki):
    _write_npc(wiki, "alpha", relations=[{"to": "beta", "kind": "союзник"}])
    _write_npc(wiki, "beta", relations=[{"to": "alpha", "kind": "союзник"}])
    data = run_lint(wiki, "--only", "bidirectional-relations")
    assert not any(i["check"] == "bidirectional-relations" for i in data["issues"])


def test_kind_asymmetry_allowed(wiki):
    """alpha says 'враг of beta', beta says 'патрон of alpha' — both are relations, direction OK"""
    _write_npc(wiki, "alpha", relations=[{"to": "beta", "kind": "враг"}])
    _write_npc(wiki, "beta", relations=[{"to": "alpha", "kind": "подчинённый"}])
    data = run_lint(wiki, "--only", "bidirectional-relations")
    # Any back-edge of ANY kind suffices
    assert not any(i["check"] == "bidirectional-relations" for i in data["issues"])
```

- [ ] **Step 2: Run tests, expect failure**

- [ ] **Step 3: Implement check**

Add to `dnd-lint.py`:

```python
@register("bidirectional-relations")
def check_bidirectional_relations(*, dnd_dir, schema, pages) -> list[Issue]:
    # Build slug → set of outbound targets
    outbound: dict[str, set[str]] = {}
    for p in pages:
        meta = p["meta"]
        own = meta.get("slug")
        if not own:
            continue
        targets = set()
        rels = meta.get("relations") or []
        if isinstance(rels, list):
            for r in rels:
                if isinstance(r, dict) and isinstance(r.get("to"), str):
                    targets.add(r["to"])
        outbound[own] = targets
    out = []
    for a, targets in outbound.items():
        for b in targets:
            if b in outbound and a not in outbound[b]:
                # Find the file for a
                afile = next((p["path"] for p in pages if p["meta"].get("slug") == a), None)
                if afile is None:
                    continue
                out.append(Issue(
                    check="bidirectional-relations", severity="warning",
                    file=afile, field="relations",
                    message=f"'{a}' → '{b}' but '{b}' has no back-edge to '{a}'",
                    fixable=False,
                ))
    return out
```

- [ ] **Step 4: Run tests, expect pass**

- [ ] **Step 5: Dry-run on real wiki**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py --only bidirectional-relations
```

Expected: several warnings on one-way relations (audit findings: eymar → liana without back-edge, first-group members, etc).

- [ ] **Step 6: Manually add reverse relations as needed**

For each warning from Step 5, Edit the target file's `relations:` list to add the reverse edge. Example:

`npcs/liana.md`:

```yaml
relations:
  - to: lander
    kind: коллега
  - to: lala-enshein
    kind: коллега
  - to: eymar
    kind: связан-с
```

Do this systematically; aim for clean.

- [ ] **Step 7: Verify lint**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py --strict
```

Expected: errors=0, warnings=0.

- [ ] **Step 8: Commit skill + data**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/scripts/dnd-lint.py skills/dnd/tests/test_dnd_lint_bidirectional.py
git commit -m "$(cat <<'EOF'
feat(lint): bidirectional-relations check (L1)

Warning-severity — any A→B relation must have a corresponding B→A
(any kind). Enforces that the graph is not one-way. Not auto-fixable
(reverse-edge kind requires semantic judgement).

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.3 L1

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

```bash
cd /Users/claw/PROJECTS/dnd-wiki
git add -A
git commit -m "fix(data): add reverse relations to close bidirectional graph

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 21: Lint check L4 — `rumour-reflected`

**Files:**
- Modify: `/Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py`
- Create: `/Users/claw/.openclaw/skills/dnd/tests/test_dnd_lint_rumour_reflected.py`

- [ ] **Step 1: Write failing tests**

Create `tests/test_dnd_lint_rumour_reflected.py`:

```python
"""Tests for rumour-reflected lint check (L4)."""
from __future__ import annotations
import json
import os
import subprocess
import sys
import textwrap
from pathlib import Path

SKILL_ROOT = Path(__file__).resolve().parent.parent
LINT = SKILL_ROOT / "scripts" / "dnd-lint.py"


def run_lint(wiki_dir: Path, *args: str) -> dict:
    env = {**os.environ, "DND_DIR": str(wiki_dir)}
    r = subprocess.run(
        [sys.executable, str(LINT), *args, "--format", "json"],
        capture_output=True, text=True, env=env,
    )
    return json.loads(r.stdout)


def test_confirmed_rumour_not_reflected_flagged(wiki):
    (wiki / "npcs" / "alpha.md").write_text(textwrap.dedent("""\
        ---
        name: A
        slug: alpha
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: нейтральный
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        ---

        body without mention of the fact
    """))
    (wiki / "rumours.md").write_text(textwrap.dedent("""\
        # Rumours

        ```yaml
        - text: "Alpha is a secret cultist"
          source: test
          status: подтверждён
          related: [alpha]
        ```
    """))
    data = run_lint(wiki, "--only", "rumour-reflected")
    assert any(
        i["check"] == "rumour-reflected" and "alpha" in i["message"]
        for i in data["issues"]
    ), data


def test_reflected_in_body_no_warning(wiki):
    (wiki / "npcs" / "alpha.md").write_text(textwrap.dedent("""\
        ---
        name: A
        slug: alpha
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: нейтральный
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        ---

        body. Alpha is a secret cultist — see article.
    """))
    (wiki / "rumours.md").write_text(textwrap.dedent("""\
        # Rumours

        ```yaml
        - text: "Alpha is a secret cultist"
          source: test
          status: подтверждён
          related: [alpha]
        ```
    """))
    data = run_lint(wiki, "--only", "rumour-reflected")
    # Even partial word match ("cultist") is enough — check uses loose substring match
    assert not any(i["check"] == "rumour-reflected" for i in data["issues"])


def test_unconfirmed_rumour_not_flagged(wiki):
    (wiki / "npcs" / "alpha.md").write_text(textwrap.dedent("""\
        ---
        name: A
        slug: alpha
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: нейтральный
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        ---

        body
    """))
    (wiki / "rumours.md").write_text(textwrap.dedent("""\
        # Rumours

        ```yaml
        - text: "Alpha is a secret cultist"
          source: test
          status: неизвестно
          related: [alpha]
        ```
    """))
    data = run_lint(wiki, "--only", "rumour-reflected")
    assert not any(i["check"] == "rumour-reflected" for i in data["issues"])
```

- [ ] **Step 2: Run tests, expect failure**

- [ ] **Step 3: Implement check**

Add to `dnd-lint.py`:

```python
def _rumour_keywords(text: str) -> list[str]:
    """Extract significant keywords from a rumour text for body substring matching."""
    # Simple heuristic: take words longer than 4 chars, lowercased
    import re as _re
    words = _re.findall(r"[а-яёa-z0-9]{5,}", text.lower())
    return words


@register("rumour-reflected")
def check_rumour_reflected(*, dnd_dir, schema, pages) -> list[Issue]:
    entities_by_slug = {p["meta"].get("slug"): p for p in pages if p["meta"].get("slug")}
    out = []
    for entry in _parse_rumours(dnd_dir):
        if entry.get("status") != "подтверждён":
            continue
        text = entry.get("text") or ""
        related = entry.get("related") or []
        if not isinstance(related, list) or not text:
            continue
        keywords = _rumour_keywords(text)
        if not keywords:
            continue
        for slug in related:
            if not isinstance(slug, str):
                continue
            target = entities_by_slug.get(slug)
            if target is None:
                continue
            body = (target["body"] or "").lower()
            # Require at least one non-trivial keyword from rumour text in body
            if not any(kw in body for kw in keywords):
                out.append(Issue(
                    check="rumour-reflected", severity="warning",
                    file=target["path"], field=None,
                    message=f"confirmed rumour '{text[:60]}...' links to '{slug}' "
                            f"but body does not reflect the fact",
                    fixable=False,
                ))
    return out
```

- [ ] **Step 4: Run tests, expect pass**

- [ ] **Step 5: Dry-run on real wiki**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py --only rumour-reflected
```

Expected: warnings on pages where confirmed rumours should be reflected but aren't. Known cases from audit: `berser-trant.md` (Братство) — Phase 3 Task 8 already added this to body; `locations/northern-keller.md` (демиплан) — may need body update; STATE.md has a derived block for confirmed_rumours (coming in Phase 7).

- [ ] **Step 6: Manually add reflection sentences where flagged**

For each warning, Edit the target file body to include the rumour fact with a `## Известные факты` header or inline mention. Keep brief.

- [ ] **Step 7: Verify lint green**

- [ ] **Step 8: Commit skill + data**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/scripts/dnd-lint.py skills/dnd/tests/test_dnd_lint_rumour_reflected.py
git commit -m "$(cat <<'EOF'
feat(lint): rumour-reflected check (L4)

Warning-severity — if rumours.md has status=подтверждён with related:
[slug], that entity's body must contain at least one significant
keyword from the rumour text. Catches facts that are confirmed but
not propagated into the entity's authoritative prose.

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.3 L4

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

```bash
cd /Users/claw/PROJECTS/dnd-wiki
git add -A
git commit -m "fix(data): reflect confirmed rumours in entity bodies

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 22: Lint check L5 — `timeline-session-parity`

**Files:**
- Modify: `/Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py`
- Create: `/Users/claw/.openclaw/skills/dnd/tests/test_dnd_lint_timeline_parity.py`

- [ ] **Step 1: Write failing tests**

Create `tests/test_dnd_lint_timeline_parity.py`:

```python
"""Tests for timeline-session-parity lint check (L5)."""
from __future__ import annotations
import json
import os
import subprocess
import sys
import textwrap
from pathlib import Path

SKILL_ROOT = Path(__file__).resolve().parent.parent
LINT = SKILL_ROOT / "scripts" / "dnd-lint.py"


def run_lint(wiki_dir: Path, *args: str) -> dict:
    env = {**os.environ, "DND_DIR": str(wiki_dir)}
    r = subprocess.run(
        [sys.executable, str(LINT), *args, "--format", "json"],
        capture_output=True, text=True, env=env,
    )
    return json.loads(r.stdout)


def test_timeline_row_without_session_flagged(wiki):
    # timeline mentions Сессия #5 but no sessions/2026-04-01.md exists
    (wiki / "timeline.md").write_text(textwrap.dedent("""\
        # Timeline

        | 2026-04-01 | #5 | Event |
    """))
    data = run_lint(wiki, "--only", "timeline-session-parity")
    assert any(
        i["check"] == "timeline-session-parity" and "#5" in i["message"]
        for i in data["issues"]
    ), data


def test_timeline_row_matches_session_no_warning(wiki):
    (wiki / "sessions" / "2026-04-01.md").write_text(textwrap.dedent("""\
        ---
        name: s
        slug: '2026-04-01'
        type: session
        number: 5
        date: 2026-04-01
        summary_one_line: test
        tags: []
        sources: []
        created: 2026-04-01
        updated: 2026-04-01
        ---

        body
    """))
    (wiki / "timeline.md").write_text(textwrap.dedent("""\
        # Timeline

        | 2026-04-01 | #5 | Event |
    """))
    data = run_lint(wiki, "--only", "timeline-session-parity")
    assert not any(i["check"] == "timeline-session-parity" for i in data["issues"])
```

- [ ] **Step 2: Run tests, expect failure**

- [ ] **Step 3: Implement check**

Add to `dnd-lint.py`:

```python
_TIMELINE_ROW_RE = re.compile(r"\|\s*(\d{4}-\d{2}-\d{2})\s*\|\s*#(\d+)\s*\|")


@register("timeline-session-parity")
def check_timeline_session_parity(*, dnd_dir, schema, pages) -> list[Issue]:
    timeline = dnd_dir / "timeline.md"
    if not timeline.exists():
        return []
    text = timeline.read_text(encoding="utf-8")
    sessions = {p["meta"].get("date"): p["meta"].get("number")
                for p in pages if p["type"] == "session"}
    out = []
    for m in _TIMELINE_ROW_RE.finditer(text):
        date_str, num_str = m.group(1), int(m.group(2))
        if date_str not in (str(d) for d in sessions.keys()):
            out.append(Issue(
                check="timeline-session-parity", severity="warning",
                file=timeline, field=None,
                message=f"timeline row '{date_str} | #{num_str}' has no matching session file",
                fixable=False,
            ))
            continue
        actual_num = sessions.get(date_str)
        if actual_num is None:
            continue
        try:
            if int(actual_num) != num_str:
                out.append(Issue(
                    check="timeline-session-parity", severity="warning",
                    file=timeline, field=None,
                    message=f"timeline says session #{num_str} on {date_str}, "
                            f"but session file has number={actual_num}",
                    fixable=False,
                ))
        except (ValueError, TypeError):
            continue
    return out
```

Find the `sessions` comparison — the date might be a `datetime.date` object (PyYAML converts ISO dates). Adjust to handle both:

```python
def _date_key(v):
    return str(v)

    sessions = {_date_key(p["meta"].get("date")): p["meta"].get("number")
                for p in pages if p["type"] == "session"}
```

- [ ] **Step 4: Run tests, expect pass**

- [ ] **Step 5: Real wiki dry-run**

Expected: may surface mismatches if timeline and session numbers drifted.

- [ ] **Step 6: Fix or resolve any warnings**

- [ ] **Step 7: Commit**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/scripts/dnd-lint.py skills/dnd/tests/test_dnd_lint_timeline_parity.py
git commit -m "$(cat <<'EOF'
feat(lint): timeline-session-parity check (L5)

Warning-severity — every 'YYYY-MM-DD | #N |' row in timeline.md
must match a sessions/YYYY-MM-DD.md file with number=N.

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.3 L5

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 23: Lint check L6 — `item-from-loot`

**Files:**
- Modify: `/Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py`
- Create: `/Users/claw/.openclaw/skills/dnd/tests/test_dnd_lint_item_from_loot.py`

- [ ] **Step 1: Write failing tests**

Create `tests/test_dnd_lint_item_from_loot.py`:

```python
"""Tests for item-from-loot lint check (L6)."""
from __future__ import annotations
import json
import os
import subprocess
import sys
import textwrap
from pathlib import Path

SKILL_ROOT = Path(__file__).resolve().parent.parent
LINT = SKILL_ROOT / "scripts" / "dnd-lint.py"


def run_lint(wiki_dir: Path, *args: str) -> dict:
    env = {**os.environ, "DND_DIR": str(wiki_dir)}
    r = subprocess.run(
        [sys.executable, str(LINT), *args, "--format", "json"],
        capture_output=True, text=True, env=env,
    )
    return json.loads(r.stdout)


def test_artifact_without_item_file_flagged(wiki):
    (wiki / "loot.md").write_text(textwrap.dedent("""\
        # Loot

        | Предмет | Тип | Где найден | Кто держит | Примечание |
        |---------|-----|------------|------------|------------|
        | Кольцо с гравировкой | артефакт | test-loc | ashan | важно |
    """))
    data = run_lint(wiki, "--only", "item-from-loot")
    assert any(
        i["check"] == "item-from-loot" and "артефакт" in i["message"]
        for i in data["issues"]
    ), data


def test_mundane_loot_not_flagged(wiki):
    (wiki / "loot.md").write_text(textwrap.dedent("""\
        # Loot

        | Предмет | Тип | Где найден | Кто держит | Примечание |
        |---------|-----|------------|------------|------------|
        | Зелье здоровья | расходник | test-loc | ashan | common |
    """))
    data = run_lint(wiki, "--only", "item-from-loot")
    # Расходник is mundane tier
    assert not any(i["check"] == "item-from-loot" for i in data["issues"])


def test_item_file_exists_no_warning(wiki):
    (wiki / "items" / "ring-test.md").write_text(textwrap.dedent("""\
        ---
        name: Кольцо с гравировкой
        slug: ring-test
        type: item
        owner: ashan
        origin: test
        rarity: rare
        type_item: артефакт
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        ---

        body
    """))
    (wiki / "loot.md").write_text(textwrap.dedent("""\
        # Loot

        | Кольцо с гравировкой | артефакт | test-loc | ashan | ok |
    """))
    # Loose match: if loot row name appears as item's `name:`, no warning
    data = run_lint(wiki, "--only", "item-from-loot")
    assert not any(i["check"] == "item-from-loot" for i in data["issues"])
```

- [ ] **Step 2: Run, expect failure**

- [ ] **Step 3: Implement check**

Add to `dnd-lint.py`:

```python
_HIGH_TIER_TYPES = {"артефакт", "реликвия", "легендарный", "ключ"}
_LOOT_ROW_RE = re.compile(
    r"^\|\s*([^|]+?)\s*\|\s*([^|]+?)\s*\|", re.M
)


@register("item-from-loot")
def check_item_from_loot(*, dnd_dir, schema, pages) -> list[Issue]:
    loot = dnd_dir / "loot.md"
    if not loot.exists():
        return []
    text = loot.read_text(encoding="utf-8")
    # Build set of item names from items/
    item_names = set()
    for p in pages:
        if p["type"] == "item":
            n = p["meta"].get("name") or ""
            if n:
                item_names.add(n.lower().strip())
    out = []
    for m in _LOOT_ROW_RE.finditer(text):
        name_cell, type_cell = m.group(1).strip(), m.group(2).strip().lower()
        # Skip header rows
        if name_cell.lower().startswith("предмет") or "----" in name_cell:
            continue
        # Strip emoji/markdown prefix for comparison
        clean_name = re.sub(r"^[^\wЀ-ӿ]+", "", name_cell).strip().lower()
        # Only flag high-tier types
        if not any(t in type_cell for t in _HIGH_TIER_TYPES):
            continue
        if clean_name and not any(clean_name in item or item in clean_name for item in item_names):
            out.append(Issue(
                check="item-from-loot", severity="warning",
                file=loot, field=None,
                message=f"loot row '{name_cell}' is {type_cell}, but no items/*.md has a matching name",
                fixable=False,
            ))
    return out
```

- [ ] **Step 4: Run tests, expect pass**

- [ ] **Step 5: Dry-run real wiki (will have warnings pre-Phase-8)**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py --only item-from-loot
```

Expected: warnings on artifacts listed in loot.md that don't yet have items/ entries (these will be resolved in Phase 8).

- [ ] **Step 6: Commit skill code only (data deferred to Phase 8)**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/scripts/dnd-lint.py skills/dnd/tests/test_dnd_lint_item_from_loot.py
git commit -m "$(cat <<'EOF'
feat(lint): item-from-loot check (L6)

Warning-severity — loot.md rows with type containing артефакт/реликвия/
легендарный/ключ must have a corresponding items/*.md entry with a
matching name. Loose substring match to allow emoji prefixes.

Pre-Phase-8 state: expected to warn on ring-mari-klodet, bandura,
elemental stones, etc. Fixed when items/ is populated.

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.3 L6

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

Note: pre-commit hook on data repo runs lint without --strict, so these warnings won't block any future data commits. Strict mode will fail until Phase 8 completes.

---

## Phase 7: sync-state expansion

Phase goal: add `generated:active_effects` and `generated:confirmed_rumours` derived blocks to `STATE.md`, extend `generated:party` with a Name column.

### Task 24: Extend sync-state to manage `active_effects` and `confirmed_rumours`

**Files:**
- Modify: `/Users/claw/.openclaw/skills/dnd/scripts/dnd-sync-state.py`
- Modify: `/Users/claw/PROJECTS/dnd-wiki/STATE.md` (add markers)
- Create: `/Users/claw/.openclaw/skills/dnd/tests/test_dnd_sync_state_expanded.py`

- [ ] **Step 1: Write failing tests**

Create `tests/test_dnd_sync_state_expanded.py`:

```python
"""Tests for active_effects and confirmed_rumours derived blocks."""
from __future__ import annotations
import importlib.util
import sys
import textwrap
from pathlib import Path

SKILL_ROOT = Path(__file__).resolve().parent.parent
sys.path.insert(0, str(SKILL_ROOT))

spec = importlib.util.spec_from_file_location(
    "dnd_sync_state", SKILL_ROOT / "scripts" / "dnd-sync-state.py"
)
mod = importlib.util.module_from_spec(spec)
spec.loader.exec_module(mod)


def test_render_active_effects_empty():
    rows = mod.collect_active_effects(chars=[])
    rendered = mod.render_active_effects_table(rows)
    assert "(нет активных эффектов)" in rendered


def test_render_active_effects_cursed_char():
    chars = [
        {"slug": "torvin", "status": "cursed",
         "active_quest_ref": "cure-mummy-rot"},
        {"slug": "ashen", "status": "healthy"},  # skipped
    ]
    rows = mod.collect_active_effects(chars=chars)
    assert len(rows) == 1
    rendered = mod.render_active_effects_table(rows)
    assert "torvin" in rendered
    assert "cursed" in rendered
    assert "cure-mummy-rot" in rendered


def test_render_confirmed_rumours(tmp_path):
    (tmp_path / "rumours.md").write_text(textwrap.dedent("""\
        # Rumours

        ```yaml
        - text: "Alpha facts"
          source: test
          status: подтверждён
          related: [alpha]

        - text: "Unclear"
          source: test
          status: неизвестно
          related: [beta]
        ```
    """))
    rendered = mod.render_confirmed_rumours(tmp_path)
    assert "Alpha facts" in rendered
    assert "Unclear" not in rendered


def test_sync_adds_active_effects_block(tmp_path):
    (tmp_path / "STATE.md").write_text(textwrap.dedent("""\
        ---
        updated: 2026-04-24
        ---

        # State

        ## Партия сейчас

        <!-- BEGIN generated:party — do not edit; run `dnd-sync-state.py` -->
        existing
        <!-- END generated:party -->

        ## Активные эффекты / угрозы

        <!-- BEGIN generated:active_effects — do not edit; run `dnd-sync-state.py` -->
        <!-- END generated:active_effects -->
    """))
    (tmp_path / "characters").mkdir()
    (tmp_path / "characters" / "torvin.md").write_text(textwrap.dedent("""\
        ---
        name: T
        slug: torvin
        type: character
        player: P
        race: Дворф
        classes: [паладин]
        level: 4
        status: cursed
        active_quest_ref: cure-mummy-rot
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        ---
    """))
    (tmp_path / "quests").mkdir()
    (tmp_path / "rumours.md").write_text("# R\n")
    chars = mod.collect_characters(tmp_path)
    mod.sync_state_active_effects(tmp_path, chars, check_only=False)
    text = (tmp_path / "STATE.md").read_text()
    assert "torvin" in text
    assert "cursed" in text
```

- [ ] **Step 2: Run tests, expect failure**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest tests/test_dnd_sync_state_expanded.py -v
```

Expected: AttributeError (functions not defined).

- [ ] **Step 3: Implement in `dnd-sync-state.py`**

Add constants near the top:

```python
STATE_ACTIVE_EFFECTS_BEGIN = "<!-- BEGIN generated:active_effects — do not edit; run `dnd-sync-state.py` -->"
STATE_ACTIVE_EFFECTS_END = "<!-- END generated:active_effects -->"
STATE_CONFIRMED_RUMOURS_BEGIN = "<!-- BEGIN generated:confirmed_rumours — do not edit; run `dnd-sync-state.py` -->"
STATE_CONFIRMED_RUMOURS_END = "<!-- END generated:confirmed_rumours -->"

ACTIVE_EFFECT_STATUSES = {"cursed", "wounded", "dying"}
```

Add functions:

```python
def collect_active_effects(chars: list[dict]) -> list[dict]:
    out = []
    for c in chars:
        status = c.get("status")
        if status in ACTIVE_EFFECT_STATUSES:
            out.append({
                "slug": c.get("slug", "?"),
                "status": status,
                "active_quest_ref": c.get("active_quest_ref") or "",
            })
    return out


def render_active_effects_table(rows: list[dict]) -> str:
    if not rows:
        return "(нет активных эффектов)"
    lines = [
        "| Персонаж | Эффект | Связанный квест |",
        "|----------|--------|-----------------|",
    ]
    for r in rows:
        lines.append(f"| {r['slug']} | {r['status']} | {r['active_quest_ref'] or '—'} |")
    return "\n".join(lines)


def render_confirmed_rumours(dnd_dir: Path) -> str:
    rumours_path = dnd_dir / "rumours.md"
    if not rumours_path.exists():
        return "(нет данных)"
    text = rumours_path.read_text(encoding="utf-8")
    confirmed = []
    for m in re.finditer(r"```yaml\n(.*?)```", text, re.S):
        try:
            data = yaml.safe_load(m.group(1))
        except yaml.YAMLError:
            continue
        if isinstance(data, list):
            for entry in data:
                if isinstance(entry, dict) and entry.get("status") == "подтверждён":
                    confirmed.append(entry)
    if not confirmed:
        return "(нет подтверждённых)"
    lines = []
    for e in confirmed:
        src = e.get("source", "?")
        lines.append(f"- {e.get('text', '?')} _(источник: {src})_")
    return "\n".join(lines)


def sync_state_active_effects(dnd_dir: Path, chars: list[dict], *, check_only: bool) -> bool:
    path = dnd_dir / "STATE.md"
    if not path.exists():
        return False
    text = path.read_text(encoding="utf-8")
    if STATE_ACTIVE_EFFECTS_BEGIN not in text:
        return False
    try:
        new_text, changed = replace_block(
            text, STATE_ACTIVE_EFFECTS_BEGIN, STATE_ACTIVE_EFFECTS_END,
            render_active_effects_table(collect_active_effects(chars)),
        )
    except ValueError:
        return False
    if changed and not check_only:
        path.write_text(new_text, encoding="utf-8")
    return changed


def sync_state_confirmed_rumours(dnd_dir: Path, *, check_only: bool) -> bool:
    path = dnd_dir / "STATE.md"
    if not path.exists():
        return False
    text = path.read_text(encoding="utf-8")
    if STATE_CONFIRMED_RUMOURS_BEGIN not in text:
        return False
    try:
        new_text, changed = replace_block(
            text, STATE_CONFIRMED_RUMOURS_BEGIN, STATE_CONFIRMED_RUMOURS_END,
            render_confirmed_rumours(dnd_dir),
        )
    except ValueError:
        return False
    if changed and not check_only:
        path.write_text(new_text, encoding="utf-8")
    return changed
```

Update `main()` to call both new sync functions and include their drift status:

```python
    chars = collect_characters(dnd_dir)
    quest_status = collect_quest_statuses(dnd_dir)
    fm_changed = sync_state_frontmatter(dnd_dir, quest_status, check_only=args.check)
    state_changed = sync_state_file(dnd_dir, chars, check_only=args.check)
    ae_changed = sync_state_active_effects(dnd_dir, chars, check_only=args.check)
    rm_changed = sync_state_confirmed_rumours(dnd_dir, check_only=args.check)
    campaign_changed = sync_campaign_file(dnd_dir, chars, check_only=args.check)
    any_drift = state_changed or campaign_changed or fm_changed or ae_changed or rm_changed
```

Also update migrate step to insert the two new marker blocks if missing. Add helper:

```python
def migrate_state_active_effects(dnd_dir: Path) -> bool:
    path = dnd_dir / "STATE.md"
    if not path.exists():
        return False
    text = path.read_text(encoding="utf-8")
    if STATE_ACTIVE_EFFECTS_BEGIN in text:
        return False
    marker = f"\n{STATE_ACTIVE_EFFECTS_BEGIN}\n\n{STATE_ACTIVE_EFFECTS_END}\n"
    if "## Активные эффекты" in text:
        idx = text.find("## Активные эффекты")
        nxt = text.find("\n## ", idx + 4)
        insert_at = nxt if nxt != -1 else len(text)
        text = text[:insert_at] + marker + text[insert_at:]
    else:
        text = text.rstrip() + "\n\n## Активные эффекты / угрозы\n" + marker
    path.write_text(text, encoding="utf-8")
    return True


def migrate_state_confirmed_rumours(dnd_dir: Path) -> bool:
    path = dnd_dir / "STATE.md"
    if not path.exists():
        return False
    text = path.read_text(encoding="utf-8")
    if STATE_CONFIRMED_RUMOURS_BEGIN in text:
        return False
    marker = f"\n{STATE_CONFIRMED_RUMOURS_BEGIN}\n\n{STATE_CONFIRMED_RUMOURS_END}\n"
    text = text.rstrip() + "\n\n## Подтверждённые слухи / факты\n" + marker
    path.write_text(text, encoding="utf-8")
    return True
```

Call them inside `main()` when `args.migrate` is set:

```python
    if args.migrate:
        s = migrate_state_file(dnd_dir)
        c = migrate_campaign_file(dnd_dir)
        ae = migrate_state_active_effects(dnd_dir)
        cr = migrate_state_confirmed_rumours(dnd_dir)
        chars = collect_characters(dnd_dir)
        sync_state_file(dnd_dir, chars, check_only=False)
        sync_campaign_file(dnd_dir, chars, check_only=False)
        sync_state_active_effects(dnd_dir, chars, check_only=False)
        sync_state_confirmed_rumours(dnd_dir, check_only=False)
        append_log(dnd_dir, "sync-state", "migrate",
                   [f"- STATE markers: {s}", f"- CAMPAIGN markers: {c}",
                    f"- active_effects markers: {ae}",
                    f"- confirmed_rumours markers: {cr}"])
        sys.exit(0)
```

Also update the `drift_checks` in `dnd-lint.py` (`state-drift` check) to call the new sync functions — replace `check_state_drift`:

```python
@register("state-drift")
def check_state_drift(*, dnd_dir, schema, pages) -> list[Issue]:
    path = dnd_dir / "STATE.md"
    if not path.exists():
        return []
    mod = _load_sync_state_module()
    if mod is None:
        return []
    chars = mod.collect_characters(dnd_dir)
    quest_status = mod.collect_quest_statuses(dnd_dir)
    try:
        body_drift = mod.sync_state_file(dnd_dir, chars, check_only=True)
        fm_drift = mod.sync_state_frontmatter(dnd_dir, quest_status, check_only=True)
        ae_drift = mod.sync_state_active_effects(dnd_dir, chars, check_only=True)
        rm_drift = mod.sync_state_confirmed_rumours(dnd_dir, check_only=True)
    except SystemExit:
        return [Issue(
            check="state-drift", severity="error",
            file=path, field=None,
            message="STATE.md missing markers; run `dnd-sync-state.py --migrate`",
            fixable=False,
        )]
    if body_drift or fm_drift or ae_drift or rm_drift:
        return [Issue(
            check="state-drift", severity="error",
            file=path, field=None,
            message="derived blocks differ from source of truth; run `dnd-sync-state.py --fix`",
            fixable=True,
        )]
    return []
```

- [ ] **Step 4: Run tests, expect pass**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest tests/test_dnd_sync_state_expanded.py -v
```

- [ ] **Step 5: Run --migrate on real STATE.md**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-sync-state.py --migrate
```

Expected: inserts the two new marker blocks and fills them; log appended.

- [ ] **Step 6: Inspect STATE.md**

```bash
cat /Users/claw/PROJECTS/dnd-wiki/STATE.md
```

Expected: new sections `## Активные эффекты / угрозы` (showing Torvin cursed + quest ref) and `## Подтверждённые слухи / факты` (Berser, демиплан, чаепитие inst).

- [ ] **Step 7: Full lint green**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py --strict
```

- [ ] **Step 8: Commit skill + data**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/scripts/dnd-sync-state.py skills/dnd/scripts/dnd-lint.py skills/dnd/tests/test_dnd_sync_state_expanded.py
git commit -m "$(cat <<'EOF'
feat(sync-state): active_effects + confirmed_rumours derived blocks

STATE.md gains two new auto-generated sections:
- generated:active_effects — sourced from characters/*.status
  ∈ {cursed, wounded, dying}, cross-referencing active_quest_ref.
- generated:confirmed_rumours — filtered from rumours.md by
  status=подтверждён.

--migrate inserts markers; --fix updates content; state-drift lint
check extended to cover both new blocks.

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.4

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

```bash
cd /Users/claw/PROJECTS/dnd-wiki
git add STATE.md log.md
git commit -m "feat(state): active_effects + confirmed_rumours generated blocks

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 25: Extend `generated:party` block with Name column

**Files:**
- Modify: `/Users/claw/.openclaw/skills/dnd/scripts/dnd-sync-state.py` (render_party_table)

- [ ] **Step 1: Modify test for party table rendering**

Append to `tests/test_dnd_sync_state.py` (existing file) or to `test_dnd_sync_state_expanded.py`:

```python
def test_render_party_table_includes_name_column():
    chars = [
        {"slug": "torvin", "name": "Торвин", "level": 4, "hp_max": 38, "status": "cursed"},
    ]
    rendered = mod.render_party_table(chars)
    assert "Торвин" in rendered
    assert "Имя" in rendered or "Name" in rendered
    assert "torvin" in rendered
```

- [ ] **Step 2: Run, expect failure**

- [ ] **Step 3: Modify `render_party_table`**

Replace in `dnd-sync-state.py`:

```python
def render_party_table(chars: list[dict]) -> str:
    lines = [
        "| Имя | slug | Уровень | HP max | Статус |",
        "|-----|------|---------|--------|--------|",
    ]
    if not chars:
        lines.append("| (no characters) | | | | |")
    for c in chars:
        name = c.get("name", "?")
        slug = c.get("slug", "?")
        level = c.get("level", "?")
        hp = c.get("hp_max", "TBD") if c.get("hp_max") is not None else "TBD"
        status = c.get("status", "?")
        lines.append(f"| {name} | {slug} | {level} | {hp} | {status} |")
    return "\n".join(lines)
```

- [ ] **Step 4: Run tests, expect pass**

- [ ] **Step 5: Run --fix on real state**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-sync-state.py --fix
```

Expected: STATE.md party table now has Имя column.

- [ ] **Step 6: Verify lint + inspect**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py --strict
head -25 /Users/claw/PROJECTS/dnd-wiki/STATE.md
```

- [ ] **Step 7: Commit skill + data**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/scripts/dnd-sync-state.py skills/dnd/tests/test_dnd_sync_state_expanded.py
git commit -m "feat(sync-state): add Имя column to party table

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"

cd /Users/claw/PROJECTS/dnd-wiki
git add STATE.md log.md
git commit -m "feat(state): regenerate party table with Имя column

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Phase 8: Items migration

Phase goal: create 8 named-item files in `items/`, trim `loot.md` to retain only non-named/mundane entries, commit.

### Task 26: Create 8 item files + trim loot.md

**Files:**
- Create: 8 files under `/Users/claw/PROJECTS/dnd-wiki/items/`
- Modify: `/Users/claw/PROJECTS/dnd-wiki/loot.md`

- [ ] **Step 1: Create `items/ring-mari-klodet.md`**

```yaml
---
name: Кольцо с гравировкой «Мари-Клодетт»
slug: ring-mari-klodet
type: item
owner: ironiya-leonberger
origin: find-ilkai
rarity: rare
type_item: артефакт
first_seen: '2026-02-14'
tags:
- кольцо
- призрак
- мари-клодет
sources: []
relations:
  - to: ironiya-leonberger
    kind: связан-с
  - to: mari-klodet
    kind: связан-с
open_questions:
  - text: "Кто такая Мари-Клодетт? Связь с партией?"
    category: lore
    asked: 2026-04-24
    answer: null
created: '2026-04-24'
updated: '2026-04-24'
---

## Описание

Кольцо с выгравированным именем **Мари-Клодетт**. Выпало при превращении призрака-Илкая в камень (Сессия 3, 2026-02-14). Добавлено в коллекцию Ирронии.

## Значимость

Неизвестна. Одно из 5 колец семьи Файрмей, собранных за сессии 2-4.
```

- [ ] **Step 2: Create `items/bandura-miala.md`**

```yaml
---
name: Бандура (Священный символ Миалы)
slug: bandura-miala
type: item
owner: ashan
origin: find-ilkai
rarity: very-rare
type_item: реликвия
first_seen: '2026-02-14'
tags:
- волынка
- миала
- храм-ильматера
relations:
  - to: ashan
    kind: связан-с
  - to: miala
    kind: связан-с
  - to: temple-ilmater
    kind: связан-с
sources: []
created: '2026-04-24'
updated: '2026-04-24'
---

## Описание

Священный символ Миалы в форме бандуры (волынки). Получена Ашен как награда от дерева Илкай (Сессия 3).

## Способности

- **Настроена на владельца** (Ашен)
- **Раз в сутки, легендарным действием:** исцелить кого-то

## Сюжетное значение

Ашен использовала бандуру в Сессии 5 — подняла Торвина после падения в бою с Маргаритой; затем играла оркестр на сожжении мумии.
```

- [ ] **Step 3-9: Create remaining 6 item files** (elemental-stone-fire, elemental-stone-air, musical-instrument-1984, hat-of-vredit, mechanical-canary, gold-pipe) — each with analogous frontmatter. Full contents:

**`items/elemental-stone-fire.md`:**

```yaml
---
name: Камень элементаля (огонь)
slug: elemental-stone-fire
type: item
owner: torvin-kamneklyatv
origin: 2026-02-14
rarity: uncommon
type_item: инструмент
first_seen: '2026-02-14'
tags:
- камень
- огонь
- академия-огненные-ладони
relations:
  - to: torvin-kamneklyatv
    kind: связан-с
sources: []
created: '2026-04-24'
updated: '2026-04-24'
---

## Описание

Приз с арены Академии магии «Огненные ладони», Сессия 3. Стихийный камень огня — однократное применение.

## Статус

В инвентаре Торвина. Не использован.
```

**`items/elemental-stone-air.md`:**

```yaml
---
name: Камень элементаля (воздух)
slug: elemental-stone-air
type: item
owner: ironiya-leonberger
origin: 2026-02-14
rarity: uncommon
type_item: инструмент
first_seen: '2026-02-14'
last_seen: '2026-03-07'
tags:
- камень
- воздух
- академия-огненные-ладони
relations:
  - to: ironiya-leonberger
    kind: связан-с
sources: []
created: '2026-04-24'
updated: '2026-04-24'
---

## Описание

Приз с арены. Стихийный камень воздуха.

## Статус

**Использован** Ирронией в Сессии 4 — воздушный элементаль добил Маргариту в бою у руин.
```

**`items/musical-instrument-1984.md`:**

```yaml
---
name: Музыкальный инструмент (1984?)
slug: musical-instrument-1984
type: item
owner: ashan
origin: temple-ilmater
rarity: rare
type_item: ключ
first_seen: '2026-01-18'
tags:
- инструмент
- храм-ильматера
- миала
relations:
  - to: ashan
    kind: связан-с
  - to: temple-ilmater
    kind: связан-с
  - to: miala
    kind: связан-с
open_questions:
  - text: "Как работает инструмент? Куда его вставить? Святилище Миалы не сработало."
    category: blocker
    asked: 2026-04-24
    answer: null
sources: []
created: '2026-04-24'
updated: '2026-04-24'
---

## Описание

Странный музыкальный инструмент, найденный в склепе храма Ильматера (Сессия 2). Название не запомнено («1984?»).

## Попытка активации

Партия попробовала вставить инструмент в святилище Миалы — **ничего не произошло**, забрали с собой.

## Гипотезы

Ключ к отпиранию какого-то ритуала или саркофага. Связан с Миалой/Алистером.

Фото: `assets/item-musical-instrument-1984.jpg`.
```

**`items/hat-of-vredit.md`:**

```yaml
---
name: Шляпа вредителей
slug: hat-of-vredit
type: item
owner: null
origin: dungeon-ruins
rarity: common
type_item: реликвия
first_seen: '2026-03-07'
tags:
- шляпа
- руины
sources: []
created: '2026-04-24'
updated: '2026-04-24'
---

## Описание

Безделушка, добытая в руинах орков (Сессия 4). «Шляпа вредителей» — назначение неясно.
```

**`items/mechanical-canary.md`:**

```yaml
---
name: Механическая канарейка в гномьей лампе
slug: mechanical-canary
type: item
owner: null
origin: dungeon-ruins
rarity: uncommon
type_item: инструмент
first_seen: '2026-03-07'
tags:
- канарейка
- гномья-лампа
sources: []
created: '2026-04-24'
updated: '2026-04-24'
---

## Описание

Механическая канарейка, помещённая внутрь гномьей лампы. Добыта в Сессии 4 (руины). Назначение неясно.
```

**`items/gold-pipe.md`:**

```yaml
---
name: Инкрустированная золотая трубка
slug: gold-pipe
type: item
owner: torvin-kamneklyatv
origin: dungeon-ruins
rarity: uncommon
type_item: инструмент
first_seen: '2026-03-28'
tags:
- трубка
- катакомбы
relations:
  - to: torvin-kamneklyatv
    kind: связан-с
  - to: lenivets
    kind: связан-с
sources: []
created: '2026-04-24'
updated: '2026-04-24'
---

## Описание

Инкрустированная золотая трубка, найденная Торвином в сокровищнице катакомб под руинами (Сессия 5). Стоимость ~25 GP.

## Канон-связь

Торвин подсел на трубку Ленивца после курения в отдыхе — судьба дала ему собственную. Шутка игрока превратилась в канон.
```

- [ ] **Step 10: Trim `loot.md`**

Remove the individual item rows from the "Активные предметы" table and Session 5 list, replacing with pointer lines like:

```markdown
| 💍 Кольцо с Мари-Клодетт | артефакт | см. [items/ring-mari-klodet.md](items/ring-mari-klodet.md) |
```

Keep generic/mundane entries (зелье здоровья, чайный набор +2 STR, документы/записки) as-is. loot.md becomes a **pointer + ordinary loot log**, not a source of truth for named items.

- [ ] **Step 11: Run full lint**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py --strict
```

Expected: errors=0. item-from-loot warnings should drop to 0 (or match remaining loot rows with proper item files).

- [ ] **Step 12: Commit data**

```bash
cd /Users/claw/PROJECTS/dnd-wiki
git add items/ loot.md log.md
git commit -m "$(cat <<'EOF'
feat(items): migrate 8 named items from loot.md to items/

- ring-mari-klodet (артефакт, Иррония)
- bandura-miala (реликвия, Ашен)
- elemental-stone-fire / elemental-stone-air (uncommon)
- musical-instrument-1984 (ключ, Ашен)
- hat-of-vredit (реликвия, trinket)
- mechanical-canary (uncommon)
- gold-pipe (uncommon, Торвин)

loot.md trimmed to mundane loot + pointers to items/.
item-from-loot lint check now green.

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.5

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Phase 9: New tooling

Phase goal: land `dnd-graph.py`, `dnd-verify-compile.py`, extend `dnd-search.py` with filters, extend `dnd-embed.py` with new namespaces.

### Task 27: `dnd-graph.py` — relations graph export

**Files:**
- Create: `/Users/claw/.openclaw/skills/dnd/scripts/dnd-graph.py`
- Create: `/Users/claw/.openclaw/skills/dnd/tests/test_dnd_graph.py`

- [ ] **Step 1: Ensure networkx is available**

```bash
python3 -c "import networkx" || pip install --user networkx
```

- [ ] **Step 2: Write failing tests**

Create `tests/test_dnd_graph.py`:

```python
"""Tests for dnd-graph.py."""
from __future__ import annotations
import json
import os
import subprocess
import sys
import textwrap
from pathlib import Path

SKILL_ROOT = Path(__file__).resolve().parent.parent
GRAPH = SKILL_ROOT / "scripts" / "dnd-graph.py"


def run_graph(wiki_dir: Path, *args: str):
    env = {**os.environ, "DND_DIR": str(wiki_dir)}
    return subprocess.run(
        [sys.executable, str(GRAPH), *args],
        capture_output=True, text=True, env=env,
    )


def _write_npc(wiki, slug, relations=None):
    content = textwrap.dedent(f"""\
        ---
        name: {slug.upper()}
        slug: {slug}
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: нейтральный
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
    """)
    if relations:
        content += "relations:\n"
        for r in relations:
            content += f"  - to: {r['to']}\n    kind: {r['kind']}\n"
    content += "---\n\nbody\n"
    (wiki / "npcs" / f"{slug}.md").write_text(content)


def test_json_format_lists_edges(wiki):
    _write_npc(wiki, "alpha", relations=[{"to": "beta", "kind": "союзник"}])
    _write_npc(wiki, "beta")
    r = run_graph(wiki, "--format", "json")
    assert r.returncode == 0, r.stderr
    data = json.loads(r.stdout)
    assert "nodes" in data and "edges" in data
    slugs = {n["slug"] for n in data["nodes"]}
    assert {"alpha", "beta"} <= slugs
    assert any(e["source"] == "alpha" and e["target"] == "beta" for e in data["edges"])


def test_find_path(wiki):
    _write_npc(wiki, "alpha", relations=[{"to": "beta", "kind": "союзник"}])
    _write_npc(wiki, "beta", relations=[{"to": "gamma", "kind": "союзник"}])
    _write_npc(wiki, "gamma")
    r = run_graph(wiki, "--find-path", "alpha", "gamma")
    assert r.returncode == 0, r.stderr
    assert "alpha" in r.stdout and "beta" in r.stdout and "gamma" in r.stdout


def test_orphans(wiki):
    _write_npc(wiki, "alpha", relations=[{"to": "beta", "kind": "союзник"}])
    _write_npc(wiki, "beta")
    _write_npc(wiki, "lonely")
    r = run_graph(wiki, "--orphans")
    assert r.returncode == 0, r.stderr
    assert "lonely" in r.stdout
    assert "alpha" not in r.stdout


def test_focus_depth_2(wiki):
    _write_npc(wiki, "alpha", relations=[{"to": "beta", "kind": "союзник"}])
    _write_npc(wiki, "beta", relations=[{"to": "gamma", "kind": "союзник"}])
    _write_npc(wiki, "gamma", relations=[{"to": "delta", "kind": "союзник"}])
    _write_npc(wiki, "delta")
    r = run_graph(wiki, "--focus", "alpha", "--depth", "2", "--format", "json")
    assert r.returncode == 0, r.stderr
    data = json.loads(r.stdout)
    slugs = {n["slug"] for n in data["nodes"]}
    assert {"alpha", "beta", "gamma"} <= slugs
    # depth=2 from alpha excludes delta
    assert "delta" not in slugs
```

- [ ] **Step 3: Run tests, expect failure**

- [ ] **Step 4: Implement `scripts/dnd-graph.py`**

```python
#!/usr/bin/env python3
"""Relations graph export — DOT, JSON, focus, orphans, find-path."""
from __future__ import annotations
import argparse
import json
import sys
from pathlib import Path

SCRIPT_DIR = Path(__file__).resolve().parent
sys.path.insert(0, str(SCRIPT_DIR.parent))
from lib.kb_common import resolve_config, read_frontmatter  # noqa: E402

import networkx as nx

TYPE_FOLDERS = {
    "npc": "npcs", "quest": "quests", "location": "locations",
    "session": "sessions", "character": "characters", "item": "items",
    "rule": "rules",
}


def build_graph(dnd_dir: Path) -> nx.DiGraph:
    g = nx.DiGraph()
    for type_name, folder in TYPE_FOLDERS.items():
        d = dnd_dir / folder
        if not d.exists():
            continue
        for p in sorted(d.glob("*.md")):
            if p.name.startswith("_"):
                continue
            meta, _ = read_frontmatter(p)
            slug = meta.get("slug")
            if not slug:
                continue
            g.add_node(slug,
                       type=type_name,
                       name=meta.get("name", slug),
                       status=meta.get("status", ""),
                       path=str(p.relative_to(dnd_dir)))
    # Now edges
    for type_name, folder in TYPE_FOLDERS.items():
        d = dnd_dir / folder
        if not d.exists():
            continue
        for p in sorted(d.glob("*.md")):
            if p.name.startswith("_"):
                continue
            meta, _ = read_frontmatter(p)
            src = meta.get("slug")
            if not src:
                continue
            # relations
            for r in meta.get("relations") or []:
                if isinstance(r, dict) and isinstance(r.get("to"), str):
                    g.add_edge(src, r["to"], kind=r.get("kind", "?"))
            # mentions (session ↔ entity)
            for m in meta.get("mentions") or []:
                if isinstance(m, str):
                    g.add_edge(src, m, kind="mentions")
    return g


def to_dot(g: nx.DiGraph) -> str:
    lines = ["digraph wiki {"]
    for n, data in g.nodes(data=True):
        label = data.get("name", n).replace('"', '\\"')
        lines.append(f'  "{n}" [label="{label}", shape=box];')
    for a, b, data in g.edges(data=True):
        kind = data.get("kind", "")
        lines.append(f'  "{a}" -> "{b}" [label="{kind}"];')
    lines.append("}")
    return "\n".join(lines)


def to_json(g: nx.DiGraph) -> str:
    return json.dumps({
        "nodes": [{"slug": n, **data} for n, data in g.nodes(data=True)],
        "edges": [{"source": a, "target": b, **data} for a, b, data in g.edges(data=True)],
    }, ensure_ascii=False, indent=2)


def subgraph_focus(g: nx.DiGraph, slug: str, depth: int) -> nx.DiGraph:
    if slug not in g:
        return nx.DiGraph()
    # BFS in undirected view for depth
    nodes = {slug}
    frontier = {slug}
    for _ in range(depth):
        nxt = set()
        for u in frontier:
            nxt.update(g.successors(u))
            nxt.update(g.predecessors(u))
        frontier = nxt - nodes
        nodes.update(frontier)
    return g.subgraph(nodes).copy()


def orphans(g: nx.DiGraph) -> list[str]:
    # Nodes with no inbound AND no outbound edges
    out = []
    for n in g.nodes():
        if g.in_degree(n) == 0 and g.out_degree(n) == 0:
            out.append(n)
    return sorted(out)


def find_path(g: nx.DiGraph, a: str, b: str) -> list[str] | None:
    if a not in g or b not in g:
        return None
    try:
        return nx.shortest_path(g.to_undirected(), a, b)
    except nx.NetworkXNoPath:
        return None


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--format", choices=["dot", "json"], default="json")
    ap.add_argument("--focus", metavar="SLUG")
    ap.add_argument("--depth", type=int, default=2)
    ap.add_argument("--orphans", action="store_true")
    ap.add_argument("--find-path", nargs=2, metavar=("A", "B"))
    ap.add_argument("--neighbors", metavar="SLUG")
    args = ap.parse_args()
    cfg = resolve_config()
    dnd_dir = cfg["dnd_dir"]
    g = build_graph(dnd_dir)

    if args.orphans:
        for n in orphans(g):
            print(n)
        return
    if args.find_path:
        p = find_path(g, *args.find_path)
        if p is None:
            print(f"No path between {args.find_path[0]} and {args.find_path[1]}")
            sys.exit(1)
        print(" → ".join(p))
        return
    if args.neighbors:
        s = args.neighbors
        if s not in g:
            sys.exit(f"slug '{s}' not in graph")
        print(f"--- {s} ---")
        print("Successors:")
        for n in g.successors(s):
            print(f"  → {n} ({g[s][n].get('kind','?')})")
        print("Predecessors:")
        for n in g.predecessors(s):
            print(f"  ← {n} ({g[n][s].get('kind','?')})")
        return

    if args.focus:
        g = subgraph_focus(g, args.focus, args.depth)

    if args.format == "dot":
        print(to_dot(g))
    else:
        print(to_json(g))


if __name__ == "__main__":
    main()
```

- [ ] **Step 5: Make executable**

```bash
chmod +x /Users/claw/.openclaw/skills/dnd/scripts/dnd-graph.py
```

- [ ] **Step 6: Run tests, expect pass**

- [ ] **Step 7: Sanity-check on real wiki**

```bash
python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-graph.py --orphans | head
python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-graph.py --focus liana --depth 2 --format dot > /tmp/liana.dot && head /tmp/liana.dot
python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-graph.py --find-path torvin-kamneklyatv eymar
```

- [ ] **Step 8: Commit**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/scripts/dnd-graph.py skills/dnd/tests/test_dnd_graph.py
git commit -m "$(cat <<'EOF'
feat: dnd-graph.py — relations graph export and queries

Commands:
  --format dot|json         export full graph
  --focus SLUG --depth N    neighborhood around a node
  --orphans                 nodes with no edges (any direction)
  --find-path A B           shortest undirected path
  --neighbors SLUG          direct successors/predecessors

Uses networkx. Dependency added to requirements.

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.8

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 28: `dnd-verify-compile.py` — raw/live vs compiled diff

**Files:**
- Create: `/Users/claw/.openclaw/skills/dnd/scripts/dnd-verify-compile.py`
- Create: `/Users/claw/.openclaw/skills/dnd/tests/test_dnd_verify_compile.py`

- [ ] **Step 1: Write failing tests**

Create `tests/test_dnd_verify_compile.py`:

```python
"""Tests for dnd-verify-compile.py."""
from __future__ import annotations
import os
import subprocess
import sys
import textwrap
from pathlib import Path

SKILL_ROOT = Path(__file__).resolve().parent.parent
VERIFY = SKILL_ROOT / "scripts" / "dnd-verify-compile.py"


def run_verify(wiki_dir: Path, *args: str):
    env = {**os.environ, "DND_DIR": str(wiki_dir)}
    return subprocess.run(
        [sys.executable, str(VERIFY), *args],
        capture_output=True, text=True, env=env,
    )


def test_matches_fully(wiki, tmp_path):
    (wiki / "raw" / "live").mkdir(parents=True, exist_ok=True)
    (wiki / "raw" / "live" / "2026-05-01.md").write_text(
        "НПС: Alpha — таверна, хорошая\n"
    )
    (wiki / "sessions" / "2026-05-01.md").write_text(textwrap.dedent("""\
        ---
        name: s
        slug: '2026-05-01'
        type: session
        number: 6
        date: 2026-05-01
        summary_one_line: test
        tags: []
        sources: []
        created: 2026-05-01
        updated: 2026-05-01
        ---

        Встретили Alpha в таверне, хороший.
    """))
    r = run_verify(wiki, "--session", "2026-05-01")
    assert r.returncode == 0
    assert "missing" not in r.stdout.lower() or "0" in r.stdout


def test_flags_missing_line(wiki):
    (wiki / "raw" / "live").mkdir(parents=True, exist_ok=True)
    (wiki / "raw" / "live" / "2026-05-01.md").write_text(
        "НПС: Alpha — таверна\n"
        "Секретная подсказка: сундук под мостом\n"
    )
    (wiki / "sessions" / "2026-05-01.md").write_text(textwrap.dedent("""\
        ---
        name: s
        slug: '2026-05-01'
        type: session
        number: 6
        date: 2026-05-01
        summary_one_line: test
        tags: []
        sources: []
        created: 2026-05-01
        updated: 2026-05-01
        ---

        Встретили Alpha в таверне.
    """))
    r = run_verify(wiki, "--session", "2026-05-01")
    assert r.returncode == 0
    assert "сундук" in r.stdout or "missing" in r.stdout.lower()
    cache = wiki / ".cache" / "verify-2026-05-01.md"
    assert cache.exists()
    assert "сундук" in cache.read_text()
```

- [ ] **Step 2: Run, expect failure**

- [ ] **Step 3: Implement `scripts/dnd-verify-compile.py`**

```python
#!/usr/bin/env python3
"""Diff raw/live/YYYY-MM-DD.md against compiled session + entities.

For each significant line in the raw note, check if a trace of it exists
in the compiled session body, timeline.md, entity body, rumours.md, or
loot.md. Unmatched lines go to .cache/verify-<date>.md for review.

Exit code always 0 — informational. Run as step 6.5 of compile ritual.
"""
from __future__ import annotations
import argparse
import re
import sys
from pathlib import Path

SCRIPT_DIR = Path(__file__).resolve().parent
sys.path.insert(0, str(SCRIPT_DIR.parent))
from lib.kb_common import resolve_config, append_log  # noqa: E402


def _significant_words(line: str) -> list[str]:
    return re.findall(r"[а-яёА-ЯЁa-zA-Z0-9]{5,}", line)


def _line_matches(line: str, haystack: str) -> bool:
    words = _significant_words(line)
    if not words:
        return True
    # Line is "matched" if at least 2 significant words (or 1 if the line has <2) appear
    haystack_lower = haystack.lower()
    hits = sum(1 for w in words if w.lower() in haystack_lower)
    needed = min(2, len(words))
    return hits >= needed


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--session", required=True, help="YYYY-MM-DD")
    args = ap.parse_args()
    cfg = resolve_config()
    dnd_dir = cfg["dnd_dir"]
    raw = dnd_dir / "raw" / "live" / f"{args.session}.md"
    if not raw.exists():
        print(f"No raw/live/{args.session}.md — nothing to verify.")
        return
    session_file = dnd_dir / "sessions" / f"{args.session}.md"
    haystack_parts = []
    if session_file.exists():
        haystack_parts.append(session_file.read_text(encoding="utf-8"))
    for extra in ["timeline.md", "rumours.md", "loot.md", "STATE.md", "questions.md"]:
        p = dnd_dir / extra
        if p.exists():
            haystack_parts.append(p.read_text(encoding="utf-8"))
    # Include all entity bodies
    for folder in ["npcs", "quests", "locations", "characters", "items"]:
        d = dnd_dir / folder
        if not d.exists():
            continue
        for fp in d.glob("*.md"):
            haystack_parts.append(fp.read_text(encoding="utf-8"))
    haystack = "\n".join(haystack_parts)

    missing = []
    for raw_line in raw.read_text(encoding="utf-8").splitlines():
        stripped = raw_line.strip()
        if not stripped or stripped.startswith("#"):
            continue
        # skip timestamp prefixes and markers
        if stripped.startswith("—") or stripped.startswith("--"):
            continue
        if len(stripped) < 8:
            continue
        if not _line_matches(stripped, haystack):
            missing.append(raw_line)

    cache_dir = dnd_dir / ".cache"
    cache_dir.mkdir(exist_ok=True)
    report = cache_dir / f"verify-{args.session}.md"
    if missing:
        report.write_text(
            f"# verify-compile {args.session}\n\n"
            f"Raw lines not found in compiled outputs ({len(missing)}):\n\n"
            + "\n".join(f"- {m}" for m in missing) + "\n"
        )
        print(f"verify-compile: {len(missing)} missing lines → {report}")
    else:
        if report.exists():
            report.unlink()
        print("verify-compile: all raw lines matched")
    append_log(dnd_dir, "note",
               f"verify-compile session={args.session} missing={len(missing)}")


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Make executable**

```bash
chmod +x /Users/claw/.openclaw/skills/dnd/scripts/dnd-verify-compile.py
```

- [ ] **Step 5: Run tests, expect pass**

- [ ] **Step 6: Add `.cache/` to `.gitignore`**

Edit `/Users/claw/PROJECTS/dnd-wiki/.gitignore`:

```
.cache/
```

- [ ] **Step 7: Commit**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/scripts/dnd-verify-compile.py skills/dnd/tests/test_dnd_verify_compile.py
git commit -m "$(cat <<'EOF'
feat: dnd-verify-compile.py — raw/live vs compiled diff

Scans raw/live/<date>.md for lines whose significant words don't
appear in the compiled session file or any entity body. Unmatched
lines → .cache/verify-<date>.md for human review.

Never halts — informational. Integrates as step 6.5 of compile ritual.

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.8

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

```bash
cd /Users/claw/PROJECTS/dnd-wiki
git add .gitignore
git commit -m "chore: ignore .cache/ (verify-compile reports)

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

### Task 29: Extend `dnd-search.py` with faceted filters

**Files:**
- Modify: `/Users/claw/.openclaw/skills/dnd/scripts/dnd-search.py`
- Create: `/Users/claw/.openclaw/skills/dnd/tests/test_dnd_search_filters.py`

- [ ] **Step 1: Read current `dnd-search.py` to understand structure**

```bash
head -80 /Users/claw/.openclaw/skills/dnd/scripts/dnd-search.py
```

- [ ] **Step 2: Write failing tests**

Create `tests/test_dnd_search_filters.py`. Skip this task's tests if ollama is unreachable. Uses a network-guard helper:

```python
"""Tests for dnd-search.py filters. Skipped when Ollama unreachable."""
from __future__ import annotations
import os
import socket
import subprocess
import sys
import textwrap
from pathlib import Path

import pytest

SKILL_ROOT = Path(__file__).resolve().parent.parent
SEARCH = SKILL_ROOT / "scripts" / "dnd-search.py"


def _ollama_up():
    try:
        socket.create_connection(("forge.lan", 11434), timeout=0.5).close()
        return True
    except OSError:
        return False


needs_ollama = pytest.mark.skipif(not _ollama_up(), reason="ollama not reachable")


def run_search(wiki_dir: Path, *args: str):
    env = {**os.environ, "DND_DIR": str(wiki_dir)}
    return subprocess.run(
        [sys.executable, str(SEARCH), *args],
        capture_output=True, text=True, env=env,
    )


@needs_ollama
def test_type_filter(wiki):
    # NPC
    (wiki / "npcs" / "alpha.md").write_text(textwrap.dedent("""\
        ---
        name: Alpha
        slug: alpha
        type: npc
        race: x
        role: x
        location: test-loc
        status: живой
        relation: нейтральный
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        ---
        Alpha living in test-loc.
    """))
    # Quest
    (wiki / "quests" / "findit.md").write_text(textwrap.dedent("""\
        ---
        name: Find It
        slug: findit
        type: quest
        status: активный
        priority: основной
        giver: alpha
        tags: []
        sources: []
        created: 2026-04-24
        updated: 2026-04-24
        ---
        Find the thing.
    """))
    # Build embeddings
    subprocess.run([sys.executable, str(SKILL_ROOT / "scripts" / "dnd-embed.py"), "--force"],
                   env={**os.environ, "DND_DIR": str(wiki)}, capture_output=True)
    r = run_search(wiki, "Alpha", "--type", "npc")
    assert r.returncode == 0, r.stderr
    assert "alpha" in r.stdout
    assert "findit" not in r.stdout
```

- [ ] **Step 3: Run tests — may skip on Ollama unavailable**

- [ ] **Step 4: Modify `dnd-search.py` to add filter arguments**

In `main()`, extend the argparse:

```python
    parser.add_argument("--type", metavar="T", choices=[
        "npc", "quest", "location", "session", "character", "item", "rule"])
    parser.add_argument("--relation", metavar="KIND")
    parser.add_argument("--status")
    parser.add_argument("--location", metavar="LOC_SLUG")
    parser.add_argument("--include-rumours", action="store_true")
    parser.add_argument("--explain", action="store_true",
                        help="Print matched snippet under each hit")
    parser.add_argument("--min-score", type=float, default=0.0)
```

Then after the embedding-based shortlist, pre-filter result rows by frontmatter checks before display:

```python
def _load_meta(dnd_dir: Path, relative_path: str) -> dict:
    from lib.kb_common import read_frontmatter
    p = dnd_dir / relative_path
    if not p.exists():
        return {}
    meta, _ = read_frontmatter(p)
    return meta


def _passes_filters(meta: dict, args) -> bool:
    if args.type and meta.get("type") != args.type:
        return False
    if args.status and str(meta.get("status", "")) != args.status:
        return False
    if args.location and str(meta.get("location", "")) != args.location:
        return False
    if args.relation:
        rels = meta.get("relations") or []
        if isinstance(rels, list):
            if not any(isinstance(r, dict) and r.get("kind") == args.relation for r in rels):
                return False
    return True
```

Wrap the result-rendering loop with this filter and the `--min-score` threshold; print snippet when `--explain` is set.

- [ ] **Step 5: Run tests, expect pass (or skip when Ollama unreachable)**

- [ ] **Step 6: Manual sanity check on real wiki**

```bash
python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-search.py "демиплан" --type location
python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-search.py "амулет" --explain --min-score 0.5
```

- [ ] **Step 7: Commit**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/scripts/dnd-search.py skills/dnd/tests/test_dnd_search_filters.py
git commit -m "$(cat <<'EOF'
feat: dnd-search.py — faceted filters and --explain

New flags:
  --type      {npc,quest,location,session,character,item,rule}
  --relation  KIND  (relations.kind must include)
  --status    STATUS
  --location  SLUG
  --min-score FLOAT
  --include-rumours
  --explain

Filters apply post-embedding to avoid recomputing. --explain prints
matched snippet under each hit.

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.8

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 30: Extend `dnd-embed.py` with rumours / timeline / questions namespaces

**Files:**
- Modify: `/Users/claw/.openclaw/skills/dnd/scripts/dnd-embed.py`

- [ ] **Step 1: Read current namespaces logic**

```bash
grep -n "namespace" /Users/claw/.openclaw/skills/dnd/scripts/dnd-embed.py | head -30
```

- [ ] **Step 2: Add `rumours`, `timeline`, `questions` to NAMESPACE_SOURCES**

Modify the structure (look for `NAMESPACES` or similar dict). Add handlers:

- `rumours` — parse `rumours.md` YAML blocks, one chunk per entry (text + source + status).
- `timeline` — chunk by major headings (`### Сессия N` or `### Предыстория` etc.), one chunk per heading.
- `questions` — entity open_questions + `questions.md` entries, one chunk per question.

If the script uses a plugin registry pattern, register three new functions. If it's an if/elif chain, extend it.

Minimal implementation sketch (add inside whatever loop exists):

```python
def _chunk_rumours(dnd_dir: Path) -> list[tuple[str, str]]:
    """(chunk_id, text) pairs."""
    out = []
    p = dnd_dir / "rumours.md"
    if not p.exists():
        return out
    text = p.read_text(encoding="utf-8")
    for i, m in enumerate(re.finditer(r"```yaml\n(.*?)```", text, re.S)):
        data = yaml.safe_load(m.group(1))
        if isinstance(data, list):
            for j, entry in enumerate(data):
                if isinstance(entry, dict) and entry.get("text"):
                    cid = f"rumour:{i}:{j}"
                    body = f"{entry.get('text')} (источник: {entry.get('source','?')}, статус: {entry.get('status','?')})"
                    out.append((cid, body))
    return out


def _chunk_timeline(dnd_dir: Path) -> list[tuple[str, str]]:
    p = dnd_dir / "timeline.md"
    if not p.exists():
        return []
    text = p.read_text(encoding="utf-8")
    # Split on "### " headings
    parts = re.split(r"\n(?=### )", text)
    out = []
    for i, part in enumerate(parts):
        first = part.splitlines()[0].lstrip("# ").strip() if part.strip() else ""
        cid = f"timeline:{i}:{_slug_safe(first)}"
        out.append((cid, part.strip()))
    return out


def _chunk_questions(dnd_dir: Path) -> list[tuple[str, str]]:
    out = []
    # Wiki-wide
    p = dnd_dir / "questions.md"
    if p.exists():
        text = p.read_text(encoding="utf-8")
        for m in re.finditer(r"```yaml\n(.*?)```", text, re.S):
            data = yaml.safe_load(m.group(1))
            if isinstance(data, list):
                for entry in data:
                    if isinstance(entry, dict) and entry.get("text"):
                        cid = entry.get("id", "?")
                        out.append((cid, entry["text"] + " (cat=" + entry.get("category","?") + ")"))
    # Entity-scoped
    for folder in ["npcs", "quests", "locations", "sessions", "characters", "items"]:
        d = dnd_dir / folder
        if not d.exists():
            continue
        for fp in d.glob("*.md"):
            if fp.name.startswith("_"):
                continue
            from lib.kb_common import read_frontmatter
            meta, _ = read_frontmatter(fp)
            for i, q in enumerate(meta.get("open_questions") or []):
                if isinstance(q, dict) and q.get("text"):
                    cid = f"{meta.get('slug','?')}#{i}"
                    out.append((cid, q["text"]))
    return out


def _slug_safe(s: str) -> str:
    return re.sub(r"[^a-zA-Zа-яА-Я0-9-]+", "-", s).strip("-")[:30]
```

Hook these into the main namespace iteration: when `--namespace rumours|timeline|questions|all` is passed, invoke the matching chunker and embed each `(cid, body)` tuple.

- [ ] **Step 3: Test by running full rebuild**

```bash
python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-embed.py --force --namespace all
```

Expected: log lines indicate embeddings added for rumours, timeline, questions.

- [ ] **Step 4: Verify via search**

```bash
python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-search.py "демиплан" --include-rumours
```

Expected: hits include the confirmed rumour entry with source attribution.

- [ ] **Step 5: Commit**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/scripts/dnd-embed.py
git commit -m "$(cat <<'EOF'
feat: dnd-embed.py — rumours, timeline, questions namespaces

Adds three new namespaces to embedding index:
- rumours: one chunk per rumours.md YAML entry
- timeline: one chunk per ### heading in timeline.md
- questions: one chunk per open_question (entity-scoped + questions.md)

--namespace all now covers everything.

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.8

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Phase 10: SKILL.md + README.md contract updates

Phase goal: update the process contract the LLM agent reads at session start so it matches the new reality (11-step ritual, single-layer knowledge, open-questions workflow, commit convention, disambiguation, update policy).

### Task 31: Rewrite `SKILL.md` with 7 new sections + 11-step ritual

**Files:**
- Modify: `/Users/claw/.openclaw/skills/dnd/SKILL.md`

- [ ] **Step 1: Read current SKILL.md**

```bash
cat /Users/claw/.openclaw/skills/dnd/SKILL.md
```

- [ ] **Step 2: Replace `## Compile ritual (9 шагов)` section with 11-step version**

Locate the existing "Compile ritual (9 шагов)" section. Replace with:

```markdown
## Compile ritual (11 шагов)

Когда пользователь говорит «структурируй» после сессии:

1. **READ** `raw/live/YYYY-MM-DD.md` + SCHEMA.md + STATE.md + index.md + last 20 log entries.
2. **PARSE** события: NPC / quest / location / loot / rumour / state-change / decision / mystery / question. Неясные элементы → category `question`.
3. **ROUTE**: existing → update (bump `updated`, append to `mentions`); new passing threshold → create (full frontmatter); below threshold → session log only; ambiguous or "не расслышал" → `open_question` в relevant entity.
4. **WRITE** `sessions/YYYY-MM-DD.md` с required frontmatter (number, date, summary_one_line, mentions, players, absent, game_day, location_ingame) + body (Что случилось / Решения / Находки / Открытые вопросы / Следующий раз / Канон-моменты).
5. **UPDATE** STATE.md (manual sections), timeline.md, rumours.md, loot.md, questions.md.
6.0. **RUN** `scripts/dnd-sync-state.py --fix` → синхронизировать derived-блоки (party + Имя, active_effects, confirmed_rumours, focus_quests).
6.3. **RUN** `scripts/dnd-rebuild.py` → index.md + .index/inbound.json.
6.5. **RUN** `scripts/dnd-verify-compile.py --session YYYY-MM-DD` → проверить что каждая значимая строка raw/live попала в compiled-вывод. Missing lines → `.cache/verify-YYYY-MM-DD.md`.
6.7. **RUN** `scripts/dnd-lint.py --fix --only body-slug-mentions,session-mentions-completeness` → backfill cross-refs.
6.9. **RUN** `scripts/dnd-lint.py --strict` → финальная валидация. Error → halt, написать частичный отчёт, ждать ручного фикса.
(Опционально) `scripts/dnd-embed.py --namespace all` → обновить vectors.db.
7. **LOG** append-only запись в log.md:
   `## [ts] compile | session YYYY-MM-DD`
   + Updated/Created/Closed/State-bump/Open-questions-added/Verify-missing.
8. **COMMIT** (только compile): `cd ${DND_DIR} && git add . && git commit -m "compile(session-N): +X npc, ~Y updated, -Z closed, ?W questions"`.
9. **REPORT** пользователю: сводка + warnings от lint (info) + commit hash + список новых open_questions (entity-path#index или q-id) + verify-compile status.
```

- [ ] **Step 3: Add new sections before `## Search backend`**

Insert these sections (one after another):

```markdown
## Knowledge layering (single-layer policy)

Вся информация в вики — знания партии. Уровни уверенности выражаются:
- **Факты** — в entity body (подтверждено).
- **Слухи** — в `rumours.md` с `status: подтверждён|неизвестно|опровергнут|в работе`.
- **Открытые вопросы** — `open_questions:` во frontmatter entity, или `questions.md` для wiki-wide.

Агент НЕ создаёт "скрытые от партии" сущности. Если ГМ-инфа ещё не раскрыта партии — её просто нет в вики.

## Open questions workflow

Агент автоматически добавляет `open_questions` entry когда при parse'е raw/live встречает маркеры неопределённости: "уточнить", "не расслышал", "не запомнил", "?", "TBD", "неизвестно" (в нон-enum полях).

Shape entry:
```yaml
  - text: "<вопрос>"
    category: clarify | lore | blocker
    asked: <today-ISO>
    asked_in: <current-session-slug>
    answer: null
```

Закрытие: `dnd-questions.py --answer <entity-slug> <index> "<answer>" [--answered-in <session>]`. Ответ переезжает в body entity (`## Известные факты`), frontmatter entry удаляется, `updated:` bump.

Wiki-wide вопросы: `questions.md`, ID формата `q-<tag>-NNN` (cal, lor, mec, ...). `dnd-questions.py --answer q-cal-001 "<answer>"` — без index.

## Disambiguation protocol

Когда raw/live содержит имя, резолвящееся в ≥2 кандидата slug'а с близкой confidence:
1. НЕ угадывать.
2. Записать ambiguity в `sessions/.../Открытые вопросы` + `aliases:` field, если имя явно вариант одного известного entity.
3. Создать `open_question` на parent-session с category=clarify.
4. Продолжить compile — обсудим с пользователем позже.

## Update conflict policy

Приоритет источника правды (highest = wins):
1. `sessions/*.md` body — канонические события.
2. `rumours.md` status=подтверждён — подтверждённый слух становится фактом.
3. Entity body — агрегированный нарратив (обновляется из 1-2).
4. `timeline.md` — краткое изложение для обзора.

Конфликт: newer session побеждает, older version уходит в `### Противоречие [YYYY-MM-DD]` блок в body entity; frontmatter `contradictions: [<other-slug>]` пополняется.

## Thresholds clarification

Создавать entity-файл:
- **NPC**: имя упомянуто 2+ раз ИЛИ central в одной сессии. НЕ создавать для one-shot encounters (гоблины сессии 1).
- **Quest**: партия взяла цель (есть giver или clear task).
- **Location**: посещена ИЛИ подробно описана 2+ раз.
- **Item**: магический / именной / quest-relevant.
- **Character**: всегда, one-per-PC.
- **Session**: всегда, one-per-session.
- **Rule**: per distinct homebrew ИЛИ referenced RAW section.
- **Rumour**: inline yaml в `rumours.md`. Отдельный quest file только когда партия действует.

Безымянные роли (торговец, стражник) остаются в session body inline. Если имя совпадает с alias уже существующего entity — halt + emit open_question.

## Commit message convention

- **compile runs**: `compile(session-N): +X npc, ~Y updated, -Z closed, ?W questions`
- **manual data fixes**: `fix: <scope> — <summary>`
- **schema changes**: `schema: <summary>`
- **skill code**: `feat:`, `feat(lint):`, `feat(sync-state):`, `feat(search):`, `feat(graph):`, `docs:`, `chore:`, `test:`

Parseable prefix; заголовок ≤ 70 символов. Body — причины и ссылки на spec/план.

## File contract (reminder)

Три категории — не меняются с v1:

| Категория | Files | Edit? |
|-----------|-------|-------|
| Source of truth | `characters/*.md`, `quests/*.md`, `npcs/*.md`, `locations/*.md`, `sessions/*.md`, `rules/*.md`, `items/*.md` | ✅ direct |
| Derived | `index.md`; `<!-- BEGIN generated:* -->` blocks в STATE/CAMPAIGN | ❌ only via scripts |
| Manual | `rumours.md`, `loot.md`, `timeline.md`, `log.md`, `questions.md`, non-derived parts of STATE/CAMPAIGN | ✅ direct |

**Никогда не редактировать между `<!-- BEGIN generated:* -->` markers.** При drift'е — `scripts/dnd-sync-state.py --fix`. `git commit --no-verify` **запрещён** — починить root cause.
```

- [ ] **Step 4: Update `## Commands` table**

Add new command rows:

```
| «вопросы / open questions / что-уточнить»         | `scripts/dnd-questions.py`        |
| «граф / visualize / связи»                         | `scripts/dnd-graph.py`            |
| «проверь компиляцию / verify-compile»              | `scripts/dnd-verify-compile.py --session …` |
```

- [ ] **Step 5: Verify SKILL.md parses as markdown (no broken YAML fences)**

```bash
grep -c '```' /Users/claw/.openclaw/skills/dnd/SKILL.md
```

Expected: even number (fences balanced).

- [ ] **Step 6: Commit**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/SKILL.md
git commit -m "$(cat <<'EOF'
docs: SKILL.md contract v2 — 11-step ritual + 7 new sections

Adds: knowledge-layering, open-questions workflow, disambiguation,
update-conflict policy, thresholds clarification, commit-message
convention, file-contract reminder. Compile ritual extended to
11 steps (verify-compile, backfill-mentions interposed).

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §C.9

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 32: Update `README.md` CLI reference + process overview

**Files:**
- Modify: `/Users/claw/PROJECTS/dnd-wiki/README.md`

- [ ] **Step 1: Add CLI entries for new scripts**

In the `## CLI reference` section, append:

```markdown
### `dnd-questions.py` — open questions workflow

```bash
dnd-questions.py                                      # list open
dnd-questions.py --by-category blocker
dnd-questions.py --by-entity <slug>
dnd-questions.py --stale [days=30]
dnd-questions.py --ask <slug> "текст" [--category clarify]
dnd-questions.py --answer <slug> <index> "<answer>" [--answered-in <session>]
dnd-questions.py --answer q-<tag>-NNN "<answer>"      # wiki-wide
```

### `dnd-graph.py` — граф связей

```bash
dnd-graph.py --format dot > /tmp/wiki.dot
dnd-graph.py --format json
dnd-graph.py --focus <slug> --depth 2
dnd-graph.py --orphans
dnd-graph.py --find-path A B
dnd-graph.py --neighbors <slug>
```

### `dnd-verify-compile.py` — проверка compile'а

```bash
dnd-verify-compile.py --session YYYY-MM-DD
```

Diff `raw/live/<date>.md` vs compiled session + entities. Unmatched lines → `.cache/verify-<date>.md`. Никогда не halts.

### `dnd-search.py` (обновлён)

Новые флаги: `--type`, `--relation`, `--status`, `--location`, `--min-score`, `--include-rumours`, `--explain`.

### `dnd-embed.py` (обновлён)

Новые namespaces: `rumours`, `timeline`, `questions`. `--namespace all` теперь покрывает всё.
```

- [ ] **Step 2: Update "Frontmatter cheatsheet"**

Add a row per type for `optional:` fields (briefly — reference SCHEMA.md for full). Example:

```markdown
**Optional fields (validated when present):**
- все типы: `aliases`, `portrait`, `map`, `open_questions`
- npc: `first_seen`, `last_seen`
- quest: `next_session`, `deadline`, `blocker`, `related_npcs`, `related_locations`
- character: `hp_max`, `subclass`, `subclass_status`, `age`, `active_quest_ref`
- item: `holder`, `type_item`, `attuned_by`, `notes`

Полный список — `SCHEMA.md` (раздел `## Optional fields per type`).
```

- [ ] **Step 3: Add "Open questions" section**

After "Жизненный цикл одной сессии":

```markdown
## Open questions — что делать если что-то непонятно

Во время сессии или после — если ГМ ещё не раскрыл деталь, или ты не расслышал — **не угадывай**. Пометь:

```bash
dnd-questions.py --ask berser-trant "Что за пояс носит?" --category clarify
dnd-questions.py --ask questions "Дата раскола Ирлина?" --id q-lor-003 --category lore
```

Перед следующей сессией:

```bash
dnd-questions.py --by-category blocker    # срочное
dnd-questions.py                           # всё открытое
```

Закрыть после ответа ГМа:

```bash
dnd-questions.py --answer berser-trant 0 "Пояс из кожи, с руной" --answered-in 2026-05-01
```

Ответ автоматически уезжает в body NPC в раздел `## Известные факты`.
```

- [ ] **Step 4: Commit**

```bash
cd /Users/claw/PROJECTS/dnd-wiki
git add README.md
git commit -m "$(cat <<'EOF'
docs: README CLI reference + open-questions workflow

Adds CLI sections for dnd-questions, dnd-graph, dnd-verify-compile;
notes new flags on dnd-search and namespaces on dnd-embed; adds
open-questions usage guide.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

### Task 33: Update pre-commit hook (remove --strict)

**Files:**
- Modify: `/Users/claw/.openclaw/skills/dnd/scripts/dnd-install-hooks.sh`

- [ ] **Step 1: Read current hook installer**

```bash
cat /Users/claw/.openclaw/skills/dnd/scripts/dnd-install-hooks.sh
```

- [ ] **Step 2: Change hook body to use non-strict lint**

Find the line that writes to the pre-commit hook — usually a heredoc. It should contain something like:

```bash
python3 "${SKILL}/scripts/dnd-lint.py" --strict
```

Change to:

```bash
python3 "${SKILL}/scripts/dnd-lint.py"
```

This keeps errors blocking but stops warnings from blocking routine commits.

- [ ] **Step 3: Re-install the hook**

```bash
bash /Users/claw/.openclaw/skills/dnd/scripts/dnd-install-hooks.sh
```

Expected output: "Pre-commit hook installed at .git/hooks/pre-commit" or similar.

- [ ] **Step 4: Verify hook content**

```bash
cat /Users/claw/PROJECTS/dnd-wiki/.git/hooks/pre-commit
```

Expected: no `--strict`.

- [ ] **Step 5: Commit hook installer change**

```bash
cd /Users/claw/.openclaw
git add skills/dnd/scripts/dnd-install-hooks.sh
git commit -m "$(cat <<'EOF'
feat(hooks): pre-commit runs lint without --strict

Severity triage: errors block commit, warnings log only. Strict mode
stays for manual runs and the 6.9 step of compile ritual.

Refs: docs/superpowers/specs/2026-04-24-dnd-wiki-hardening-v2-design.md §E

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Phase 11: Final smoke + embedding rebuild

### Task 34: Full pipeline smoke test + embeddings rebuild

**Files:** none modified; verifies full system end-to-end.

- [ ] **Step 1: Run full test suite**

```bash
cd /Users/claw/.openclaw/skills/dnd && python -m pytest -v
```

Expected: all tests pass. Previous count (52) + all new tests from Tasks 2, 4, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 27, 28, 29. Goal: 90+ tests, all green.

- [ ] **Step 2: Run full lint --strict on real wiki**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-lint.py --strict
```

Expected: `errors=0, warnings=0`. `info` messages are acceptable.

- [ ] **Step 3: Run sync-state --check**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-sync-state.py --check; echo "exit=$?"
```

Expected: `exit=0`.

- [ ] **Step 4: Rebuild index**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-rebuild.py
```

Expected: `Rebuilt index.md — N pages: ...` with items=8, NPCs ≥33, quests ≥22, locations ≥7.

- [ ] **Step 5: Rebuild embeddings**

```bash
cd /Users/claw/PROJECTS/dnd-wiki && python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-embed.py --force --namespace all
```

Expected: embeddings built across campaign, rules, rumours, timeline, questions namespaces. If Ollama unreachable — note and skip; not a blocker.

- [ ] **Step 6: Smoke test CLI scripts**

```bash
python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-summary.py
python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-questions.py
python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-graph.py --orphans
python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-graph.py --find-path torvin-kamneklyatv eymar
python3 /Users/claw/.openclaw/skills/dnd/scripts/dnd-search.py "демиплан" --include-rumours 2>/dev/null || echo "search skipped (ollama)"
```

Expected: each command completes without error.

- [ ] **Step 7: Verify success criteria from spec §Success criteria**

Manually walk through:
1. `dnd-lint.py --strict` → 0 errors, 0 warnings ✓
2. `dnd-sync-state.py --check` → exit 0 ✓
3. Every NPC has `mentions:` where applicable ✓
4. Every session has `mentions` covering body entities ✓
5. `items/` ≥8 entries ✓
6. `questions.md` + ≥3 open_questions across wiki ✓
7. `dnd-graph.py --orphans` mostly empty (or just session-only entities) ✓
8. `dnd-questions.py --ask` / `--answer` cycle works ✓
9. Pre-commit hook blocks error, allows warning ✓
10. Tests green ✓
11. Synthetic compile smoke: create a dummy raw/live, parse it, verify no halt — do manually:
    - Write `/tmp/raw-test.md` to `raw/live/2026-05-01.md` with ~10 varied lines.
    - Walk through ritual steps 6.0-6.9 commands sequentially.
    - Confirm each step completes; .cache report generated if needed.
    - Restore working state (delete test raw + revert).

- [ ] **Step 8: Final summary commit**

Nothing to commit at this point unless index.md or log.md changed from rebuild. Bundle them if they did:

```bash
cd /Users/claw/PROJECTS/dnd-wiki
git status
git add -A
git commit -m "$(cat <<'EOF'
chore: wiki hardening v2 complete — final smoke green

- 24 lint checks green on --strict
- sync-state --check clean
- index.md rebuilt
- embeddings rebuilt (campaign + rules + rumours + timeline + questions)
- All 90+ tests pass
- 11-step compile ritual validated via dry-run
- Success criteria §Success all satisfied

See: docs/superpowers/plans/2026-04-24-dnd-wiki-hardening-v2.md

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

If nothing to commit (clean tree), print confirmation instead:

```bash
echo "Phase 11 complete: wiki hardening v2 landed. No commit needed (tree clean)."
```

---

## Self-review checklist (run by author before handoff)

1. **Spec coverage** — every spec section (§C.1 through §C.10, plus Data Flow, Error Handling, Testing) has at least one task. Open questions §C.7 → Tasks 12-16. Items §C.5 → Task 26. Data fixes §C.6 → Tasks 6-9. Mentions backfill → Task 18-19 (via lint --fix). `dnd-contradict.py` is explicitly out of scope (spec says defer to v3). No gaps.

2. **Placeholder scan** — all code blocks contain concrete implementations. No "TBD" or "TODO" text. Paths are absolute. Commit messages are literal. Expected outputs are stated.

3. **Type consistency** —
   - `relations:` format used throughout: `list[{to: str, kind: str}]`. ✓
   - `open_questions:` entry shape: `{text, category, asked, answer, [asked_in], [answered_in]}`. ✓
   - `active_quest_ref:` (not `active_quest`) — character optional field. ✓
   - Enum names: `relation.kind`, `item.type_item` (not `item.type` — conflicts with common `type:`), `character.subclass_status`, `open_question.category`, `open_question.status`. ✓

4. **Testing coverage** — Tasks 2, 3, 4, 13-23, 24, 25, 27-29 all include tests. Regression: full `pytest` in Task 34. Ollama-dependent test (Task 29) uses `needs_ollama` skip.

5. **Commit discipline** — two repos, never mixed. Skill code commits `cd ~/.openclaw` with explicit paths; data commits `cd ~/PROJECTS/dnd-wiki`. Each phase ends with a clean tree.

6. **Task-ordering dependencies** — Task 2 (parse_schema optional) blocks L9 (Task 17). Task 3 (unified relations in _iter_refs) blocks Task 5 (real migration). Task 4 (migration script) blocks Task 5. L2/L3 (Tasks 18-19) must precede Task 34 Step 11 (synthetic compile). Tasks order reflects this.

---

## Handoff

Plan complete — 34 tasks across 11 phases (Phase 0 was the spec commit, already done). Two execution options:

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration. Two-stage review (worker + reviewer per task).

**2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints for review.

Which approach?

