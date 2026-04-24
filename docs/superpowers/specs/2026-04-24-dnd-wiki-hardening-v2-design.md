# D&D Wiki Hardening v2 — Design Spec

**Date:** 2026-04-24
**Status:** Approved (pending user spec review)
**Scope:** Second-pass hardening of the `dnd` skill. v1 (spec `2026-04-24-dnd-skill-hardening-design.md`) established the derived-block contract, fixed slug drift in `STATE.md`/`CAMPAIGN.md`, and ensured `lint --strict` passes on the current wiki. v2 closes the remaining gaps: incomplete SCHEMA contract, un-unified relations format, missing cross-ref checks, empty `items/`, under-used `rumours.md`, and the absence of a TODO/open-questions workflow. Goal: the LLM agent that operates this wiki cannot introduce drift, cannot miss cross-refs, cannot invent undocumented fields, and cannot silently drop uncertain information.

## Motivation

A full audit (see commit log + session transcript 2026-04-24) surfaced **26 concrete defects** across the current wiki and **9 gaps in lint coverage**. Key findings:

- **Contract gap.** ~17 frontmatter fields are actively used but not in `SCHEMA.md` — `next_session`, `deadline`, `blocker`, `first_seen`, `last_seen`, `portrait`, `hp_max`, `subclass`, `secrets`, `aliases`, `created_session`, `created_date`, `game_day`, `location_ingame`, `players`, `absent`, `related_npcs`. The agent cannot know which are canonical vs. which are drift.
- **Relations fragmentation.** Three formats in circulation: list-of-dicts with `{npc, type}` (most NPCs), list-of-dicts with `type:` as a relation-kind label (lint treats this specially), flat `related_npcs: [slug]` (locations). LLM queries against the graph are unreliable.
- **Cross-ref one-way street.** 0 of 33 NPCs carry a `mentions:` field; sessions list entities but the reverse backlink never lands. `body-slug-mentions` and `session-mentions-completeness` checks do not exist.
- **Empty items/.** 8+ magical/named/quest-relevant items live only in `loot.md` tables. The item graph is blank. Quest triggers (e.g. Bandura of Miala, rings of Mari-Clodet) are unlinkable.
- **Undocumented field drift.** `berser-trant.md` has `secrets:`, `rouen.md` has `portrait:`, several sessions have `game_day: "4-5"` (string). None validated.
- **No TODO workflow.** During live play the user sometimes mishears or cannot recall a fact. Current wiki has no structured way to mark "ask GM" — the information is silently lost or stored as freeform prose.
- **Taxonomy semantic smear.** `cities:` contains dungeons, shrines, labs, altars; `events:` contains `все-локации` and `катастрас`. Tags stop being a navigation aid.
- **Calendar inconsistency.** `STATE.md` says `день ~47`, `timeline.md` last session ends on day 17. Open question, not yet resolved.
- **Data defects (batch).** `main-luskan.giver` is a date, `ship-irlin-forest.location` is Cyrillic text, `statya-irvin` is completed-but-priority-срочно, `mystery-mage.jpg` sits in `npcs/`, 4 of 5 PC have no `hp_max`, 2 of 5 PC have subclass placeholder (`уточнить`, `не выбрана`).

v1 prevented the wiki from breaking. v2 makes it *complete*.

## Non-Goals

- Multi-campaign features beyond existing `DND_DIR` portability.
- GM-only / knowledge-layering infrastructure. Decision confirmed with user: **every fact in the wiki is party knowledge.** Uncertainty is expressed through `rumour.status` (подтверждён/опровергнут/неизвестно/в работе) and `open_questions` — not through visibility flags.
- Web UI for graph visualization (DOT/JSON export is enough for this phase).
- Migration of `rules/` SRD files or homebrew ruleset.
- Breaking changes to `DND_DIR` resolution or CLI entrypoints.
- CI/GitHub Actions setup.

## Architecture

Six layers, top-down:

1. **Contract layer** — `SCHEMA.md` (machine-readable blocks parsed by `lib/kb_common.parse_schema`) + `SKILL.md` (process contract for the LLM agent).
2. **Data layer** — entity files (`npcs/`, `quests/`, `locations/`, `sessions/`, `characters/`, `items/`, `rules/`), manual files (`rumours.md`, `loot.md`, `timeline.md`, `log.md`, new `questions.md`).
3. **Validation layer** — `dnd-lint.py` with 24 checks (15 existing + 9 new). Severity triage: `error` blocks commit, `warning` reports only, `info` logs.
4. **Derivation layer** — `dnd-sync-state.py` with expanded scope (adds `active_effects`, `confirmed_rumours`, party+names), `dnd-rebuild.py` with reverse-index `.index/inbound.json`.
5. **Query / tooling layer** — `dnd-search.py` with faceted filters, new `dnd-graph.py`, new `dnd-questions.py`, new `dnd-verify-compile.py`, `dnd-embed.py` with expanded namespace scope.
6. **Process layer** — updated `SKILL.md` (11-step compile ritual, disambiguation protocol, single-layer knowledge policy, commit-message convention, open-questions workflow).

Invariant: no layer knows about a layer above it. Schema is parsed; scripts consume parse results; the agent reads `SKILL.md` and obeys; the pre-commit hook enforces exit codes.

## Components

### C.1 SCHEMA.md — full contract documentation

Add machine-readable blocks for every field actually in use:

```
<!-- schema:optional:common -->
- aliases: list[str]          # disambiguation — alternate names
- portrait: path              # relative to repo root
- map: path                   # relative to repo root
- open_questions: list[dict]  # see open_question schema
<!-- /schema:optional:common -->

<!-- schema:optional:npc -->
- first_seen: session-slug
- last_seen: session-slug
<!-- /schema:optional:npc -->

<!-- schema:optional:quest -->
- next_session: bool                    # include in pre-session brief
- deadline: iso-date | null
- blocker: string | null
- related_npcs: list[slug]              # legacy — prefer relations:
- related_locations: list[slug]
- created_session: int
- created_date: iso-date
<!-- /schema:optional:quest -->

<!-- schema:optional:character -->
- hp_max: int
- subclass: string
- subclass_status: enum[chosen, tbd]    # NEW — makes "уточнить" machine-trackable
- age: int | string
- portrait: path
<!-- /schema:optional:character -->

<!-- schema:required:session -->  (extend existing)
- number, date, summary_one_line
- mentions: list[slug]                  # PROMOTED to required
- players: list[str]                    # NEW required
- absent: list[str]                     # NEW required (may be empty)
- game_day: string                      # regex ^\d+(-\d+)?$
- location_ingame: string               # narrative, not a slug
<!-- /schema:required:session -->

<!-- schema:required:rumour -->  (NEW)
- text: string
- source: string
- status: enum (existing)
- related: list[slug]
<!-- /schema:required:rumour -->

<!-- schema:enum:item:type -->  (NEW)
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
<!-- /schema:enum:item:type -->

<!-- schema:enum:character:subclass_status -->  (NEW)
- chosen
- tbd
<!-- /schema:enum:character:subclass_status -->

<!-- schema:enum:open_question:category -->  (NEW)
- clarify      # ask GM for detail
- lore         # deep-world question
- blocker      # blocks party progress
<!-- /schema:enum:open_question:category -->

<!-- schema:enum:open_question:status -->  (NEW)
- open
- answered
- obsolete
<!-- /schema:enum:open_question:status -->

<!-- schema:enum:relation:kind -->  (NEW — unifies all relations)
- союзник
- враг
- нейтральный
- коллега
- муж / жена
- брат / сестра
- отец / мать / сын / дочь
- наставник / ученик
- член-группы
- знакомый
- подозреваемый
- жертва
- патрон / подчинённый
- связан-с                   # catch-all for soft links
<!-- /schema:enum:relation:kind -->
```

**Unified relations format** (replaces 3 current shapes):

```yaml
relations:
  - to: <slug>
    kind: <enum:relation.kind>
```

`related_npcs:` remains as a legacy field on locations (it continues to work for broken-ref check) but is documented as deprecated; new code uses `relations`.

**Open question schema:**

```yaml
open_questions:
  - text: "Как выглядит Берсер Трант?"
    category: clarify
    asked: 2026-04-24
    asked_in: 2026-03-28          # session-slug, optional
    answer: null                   # filled by dnd-questions.py --answer
    answered_in: null              # session-slug, optional
```

### C.2 Taxonomy cleanup

Reorganize `<!-- schema:taxonomy -->` block:

- Rename `cities:` → `places:`. Split into subcategories inside the taxonomy block: `settlements`, `landmarks`, `dungeons`, `shrines`, `buildings`, `regions_of_place`.
- Split `events:` → `mechanics:` (mummy-rot, ментальная-атака, обелиски, жаровни, электричество, etc.) and `concepts:` (катастрас, прорицание, демиплан, договор-души).
- Remove near-duplicates flagged by `taxonomy-hygiene` info: `призраки` (keep `призрак`), `горный-лесник` (keep `голодный-лесник` if that's the intent, else reverse — verify before cleanup).
- Migrate existing page tags to the new groups. This is pure rename — no semantic change expected.

### C.3 Lint — 9 new checks

Each check registered via `@register(...)` in `dnd-lint.py`, following existing pattern.

| ID | Name | Severity | Fixable | Purpose |
|----|------|----------|---------|---------|
| L1 | `bidirectional-relations` | warning | no | For every `A.relations[kind].to=B`, require `B.relations` to reference `A` (any kind). Flags one-way links. |
| L2 | `body-slug-mentions` | warning | yes | Scan entity body for Cyrillic names from `npcs-named` taxonomy or explicit `[name](path/slug.md)` links. If slug is not present in `relations`/`mentions`/sources → warning. `--fix` auto-appends to `mentions`. |
| L3 | `session-mentions-completeness` | error | no | For each session.mentions[slug], require entity.mentions to contain the session slug (bidirectional). Errors if one-way. Blocks commits. |
| L4 | `rumour-reflected` | warning | no | For every rumour with `status: подтверждён` and `related: [slug]`, require the related entity body to contain a cross-reference (link or explicit mention with slug). |
| L5 | `timeline-session-parity` | warning | no | For every `## Сессия #N` header in `timeline.md`, require a matching `sessions/YYYY-MM-DD.md` with the same `number`. Warning on mismatch. |
| L6 | `item-from-loot` | warning | no | Parse `loot.md` tables; for entries with `rarity` ≥ uncommon or explicit `name:` prefix, require an `items/<slug>.md` file. |
| L7 | `unresolved-placeholders` | warning (error in `--strict`) | no | Scan frontmatter values for placeholder tokens: `TBD`, `уточнить`, `не выбрана`, `?`, `неизвестно` (only when in non-enum fields or when enum explicitly allows `неизвестно` — match case-insensitively). Flag with suggestion to move the value to `open_questions`. |
| L8 | `stale-open-question` | info | no | For every `open_questions` entry, if `asked < today - 30d` and `answer is null`, emit info. |
| L9 | `schema-undeclared-fields` | warning | no | Collect the set of known fields (`required` + `optional` per type + common). Any frontmatter key not in this set → warning. This catches invented fields like `secrets:`. |

Implementation order inside the roll-out plan: L7 and L8 land in Phase 5 (they are the open-questions infrastructure). Phase 6 delivers the rest: L9 first (cheap, catches schema drift immediately), then L2 and L3 (core graph integrity, enables the compile-ritual 6.7 backfill step), then L1, L4, L5, L6.

### C.4 sync-state.py — expanded derived blocks

Add two new derived blocks to `STATE.md`:

```markdown
## Активные эффекты / угрозы

<!-- BEGIN generated:active_effects — do not edit; run `dnd-sync-state.py` -->
| Персонаж | Эффект | Источник | Квест |
|----------|--------|----------|-------|
| torvin-kamneklyatv | cursed (Mummy Rot) | sessions/2026-03-28.md | cure-mummy-rot |
<!-- END generated:active_effects -->

## Подтверждённые слухи / факты

<!-- BEGIN generated:confirmed_rumours — do not edit; run `dnd-sync-state.py` -->
- Берсер Трант — член Братства взирающих очей (газета, сессия 2)
- Северный Келлер находится в демиплане (инсайт Торвина, сессия 5)
<!-- END generated:confirmed_rumours -->
```

Derivation:
- `active_effects`: iterate `characters/*.md` where `status ∈ {cursed, wounded, dying}`; cross-reference to quest via a new optional character field `active_quest_ref: quest-slug`. Missing reference → empty column, info-level lint flag.
- `confirmed_rumours`: parse `rumours.md`, filter `status == подтверждён`, render as bullet list with source.

Extend existing `generated:party` block: add `Name` column before `slug` for human readability.

### C.5 Items migration (selective)

Create `items/*.md` for the following 8 items (full frontmatter with `type: item`, `name`, `slug`, `owner`, `origin`, `rarity`, `type` enum, `tags`, `relations`, `sources`):

1. `ring-mari-klodet.md` — Кольцо Мари-Клодетт (артефакт, holder: ironiya-leonberger, origin: illa-tree ghost sessions/2026-02-14)
2. `bandura-miala.md` — Бандура (священный символ Миалы) (артефакт, holder: ashan, origin: дерево Илкай sessions/2026-02-14)
3. `elemental-stone-fire.md` — Камень элементаля (огонь) (uncommon, holder: torvin-kamneklyatv)
4. `elemental-stone-air.md` — Камень элементаля (воздух) (uncommon, использован sessions/2026-03-07)
5. `musical-instrument-1984.md` — Музыкальный инструмент (quest-relevant, holder: ashan, origin: святилище Миалы temple-ilmater)
6. `hat-of-vredit.md` — Шляпа вредителей (uncommon trinket, origin: руины орков sessions/2026-03-07)
7. `mechanical-canary.md` — Механическая канарейка (uncommon, в гномьей лампе)
8. `gold-pipe.md` — Инкрустированная золотая трубка (25 GP, common-uncommon) (holder: torvin-kamneklyatv)

Additional candidates to decide during migration (inspection of loot.md may surface more): chain set (+2 STR), health potion, article about Berser, soul belts (likely a rules file, not an item file — see C.8).

`loot.md` retains: session batches of ordinary loot (gold, rations, generic trinkets), cash balance notes, consumable tracker.

### C.6 Data fixes batch (from audit §1)

Single commit `fix: data integrity — 17 issues`. Ordered list:

1. `quests/main-luskan.md` — `giver: null` + open_question `{text: "Кто formally gave the Luskan quest?", category: clarify}`.
2. `quests/ship-irlin-forest.md` — `location: irlin-forest` (replace Cyrillic).
3. `quests/stone-bee.md` — `location: northern-keller` (best inference; re-evaluate if wrong).
4. `quests/statya-irvin.md` — remove `priority: срочно` (completed; priority is meaningless or set to `побочный`).
5. `npcs/mystery-mage.jpg` → `assets/npcs/mystery-mage.jpg`; update `npcs/eymar.md` body reference path.
6. `npcs/berser-trant.md` — remove `secrets:` field; add body section `## Тайная сторона` with content; add tag `братство-взирающих-очей` (after adding it to taxonomy `factions:`, already present).
7. `characters/torvin-kamneklyatv.md` — add `active_quest_ref: cure-mummy-rot`.
8-11. Characters without `hp_max` (ashan, chad-ridley, ironiya-leonberger, dimitrina-speiskaya) — attempt to fill from character sheets; if unknown, keep `hp_max: null` and add open_question per character.
12-13. Characters with placeholder subclass (dimitrina, ironiya) — convert `subclass` freeform string to `subclass: "не выбрана"/"уточнить"` replaced by `subclass: null` + `subclass_status: tbd` + `open_questions` entry.
14. `rouen.md` — `portrait:` stays (once documented in SCHEMA it's valid).
15. `characters/_party.md` — delete. Replace with a symlink-semantic note inside `characters/` describing "see ../CAMPAIGN.md for the source-of-truth table". The prose note about "Коля психанул 14.02" moves to `characters/chad-ridley.md` body.
16. Sessions `mentions:` backfill via `dnd-lint.py --fix` after L2/L3 are implemented.
17. All NPCs — backfill `mentions: [session-slug, ...]` via the same lint fix pass.

### C.7 Open questions infrastructure

**Files:**
- `questions.md` — root for wiki-wide questions not tied to one entity (calendar discrepancy, pre-campaign history, mechanics).
- `open_questions:` in every entity frontmatter (optional, list of dicts).

**Script `dnd-questions.py`:**

```
dnd-questions.py                          # list all open, grouped by entity
dnd-questions.py --by-category blocker    # filter
dnd-questions.py --by-entity berser-trant
dnd-questions.py --stale [days=30]        # only old ones
dnd-questions.py --count                  # summary numbers
dnd-questions.py --ask <slug> "<question>" [--category clarify]
                                          # add entry interactively
dnd-questions.py --answer <entity-slug> <index> "<answer>"
                                          # close question; move answer to body;
                                          # bump updated; optional --answered-in session
dnd-questions.py --export markdown        # pre-session brief format
```

Lint integration: L7 (`unresolved-placeholders`) suggests using `--ask` when it finds `TBD`/`уточнить` tokens.

Compile-ritual integration: step 3 (ROUTE) allows the agent to emit `open_question` entries when parsing live notes contains "не запомнил", "не расслышал", "уточнить", etc. These land in the relevant entity's frontmatter on first write.

**`questions.md` format** (YAML-fenced, same as rumours.md):

```yaml
- id: q-cal-001
  text: "Почему STATE.date_ingame=день ~47, а timeline.md last session заканчивается днём 17?"
  category: clarify
  asked: 2026-04-24
  answer: null
```

IDs for wiki-wide questions use format `q-<tag>-NNN` for human-assigned grouping (cal = calendar, lor = lore, mec = mechanics).

### C.8 New scripts

**`dnd-graph.py`:**
```
dnd-graph.py --format dot > graph.dot
dnd-graph.py --format json
dnd-graph.py --focus <slug> [--depth 2]
dnd-graph.py --orphans                    # pages with no inbound edges
dnd-graph.py --find-path A B              # shortest path A → B through relations
dnd-graph.py --neighbors <slug>           # direct edges only
```
Parses frontmatter of all pages; builds a directed graph (edges from `relations.to`, `mentions`, `related_*`). Uses `networkx` (add to `requirements.txt`).

**`dnd-verify-compile.py`:**
```
dnd-verify-compile.py --session 2026-05-01
```
Diff `raw/live/2026-05-01.md` vs all files modified in the same git session: newly created entities, edited session file, edited STATE/timeline/loot/rumours. Parses raw into tagged lines (`NPC:`, `QUEST:`, `LOOT:`, `RUMOUR:`, freeform). For each raw line, greps the compiled outputs. If unmatched, writes a report to `.cache/verify-<date>.md` with the missing lines. Exit 0 always (informational; compile shouldn't block). Runs in compile ritual step 6.5.

**Updated `dnd-search.py`:**
```
dnd-search.py "query" [--type npc] [--relation союзник] [--status живой]
              [--location northern-keller] [--min-score 0.65]
              [--include-rumours] [--explain]
```
`--explain` renders the matched snippet under each hit.

**Updated `dnd-embed.py`:**
New namespaces in addition to `campaign`/`rules`:
- `rumours` (each rumour entry indexed separately)
- `timeline` (grouped by year/session)
- `questions` (open_questions across entities + questions.md)

### C.9 SKILL.md updates

Add sections:

1. **Single-layer knowledge policy.** Every fact in the wiki is party knowledge. Uncertainty via `rumours.md` (text, status) and `open_questions` — never via hidden fields.
2. **Disambiguation protocol.** If a mentioned name resolves to ≥2 candidate slugs with comparable match confidence, the agent MUST NOT guess. Record the ambiguity as an `open_question` on the parent session, add `aliases:` if the name is clearly a variant of one known entity, and ask the user on next interaction.
3. **Update conflict policy.** Priority order (highest = source of truth): `sessions/*.md` body → `rumours.md` status=подтверждён → entity body → `timeline.md`. Conflict resolution: newer session wins, older session becomes `### Противоречие [date]` block in entity body, `contradictions:` frontmatter list appended.
4. **Open-questions workflow.** When to emit (live-note mentions "не расслышал"/"уточнить"); when to close (`dnd-questions.py --answer`); lint signals stale entries.
5. **Compile ritual v2** — 11 steps (see § D. Data Flow).
6. **Commit message convention** — `compile(session-N): +Xnpc, ~Y updated, -Z closed, ?W questions` for compile runs; `fix:`, `schema:`, `feat:` prefixes for hardening work.
7. **Thresholds clarification** — expand existing thresholds with explicit anti-rules: "Do not create NPC entries for one-shot encounters"; "Unnamed roles (торговец, стражник) stay in session body"; "If name collides with existing alias → halt, emit open_question".

### C.10 Reverse-index generation

`dnd-rebuild.py` extended to produce `.index/inbound.json`:

```json
{
  "liana": ["temple-ilmater", "cure-mummy-rot", "2026-01-18", ...],
  "eymar": ["liana", "2026-03-28", "investigate-eymar"],
  ...
}
```

Used by `orphan`, `bidirectional-relations`, and `body-slug-mentions` checks to avoid O(N²) re-scanning.

## Data Flow

### Compile ritual v2 (11 steps)

```
1. READ
   raw/live/YYYY-MM-DD.md
 + SCHEMA.md (parse — for types, enums, required, optional)
 + STATE.md (current party, active effects, focus)
 + index.md (what entities exist)
 + last 20 log.md entries

2. PARSE
   Extract events by category: {npc, quest, location, loot, rumour,
   state-change, decision, mystery, question}. Unclear items default to
   category=question.

3. ROUTE
   For each event:
   - existing entity → update (bump updated, append mentions)
   - new + passes threshold → create (full frontmatter per SCHEMA)
   - below threshold → session body only
   - ambiguous/mishear → open_question on the parent entity

4. WRITE sessions/YYYY-MM-DD.md
   - required frontmatter: number, date, summary_one_line, mentions,
     players, absent, game_day, location_ingame
   - body sections: Что случилось / Решения / Находки / Открытые вопросы /
     Следующий раз / Канон-моменты

5. UPDATE
   STATE.md (manual body sections) / timeline.md / rumours.md / loot.md /
   questions.md

6. DERIVE & VALIDATE (pipeline)
   6.0 dnd-sync-state.py --fix      # party, active_effects, confirmed_rumours
   6.3 dnd-rebuild.py               # index.md + .index/inbound.json
   6.5 dnd-verify-compile.py        # raw/live vs compiled diff → .cache
   6.7 dnd-lint.py --fix --only body-slug-mentions,session-mentions-completeness
                                    # backfill cross-refs
   6.9 dnd-lint.py --strict         # final validation; error → halt

   (optional) dnd-embed.py --namespace all

7. LOG
   append-only entry in log.md:
     ## [ts] compile | session YYYY-MM-DD
     - Updated: N entities
     - Created: N entities
     - Closed: N quests
     - State-bumps: active_effects/confirmed_rumours
     - Open questions added: N
     - Verify-compile: <ok | N missing lines in .cache/verify-<date>.md>

8. COMMIT (scripted)
   git add .
   git commit -m "compile(session-N): +X npc, ~Y updated, -Z closed, ?W questions"

9. REPORT to user
   - Summary of changes
   - Lint warnings (ignored because not errors)
   - Open questions newly added (with IDs / entity paths)
   - Verify-compile status
   - Commit hash
```

### Open-question lifecycle

```
created by agent (parse unclear entry)
   │
   ▼
entity.open_questions[] or questions.md
   │
   ▼ (user runs dnd-questions.py --answer <slug> <idx> "answer" [--answered-in session])
answer written into entity body (e.g. "## Известные факты" gets a line)
frontmatter entry removed (decision: rely on git history for audit)
entity.updated bumped
   │
   ▼ (lint L8 after 30 days with no answer)
info warning "stale open_question"
```

Decision on lifecycle for answered entries: **remove from frontmatter, rely on git history for audit.** Rationale: frontmatter should reflect current open work; history is git's job. Re-ask detection is not a goal here.

### Open-question identification

- **Entity-scoped** open_questions (frontmatter list on NPC/quest/location/session/character/item) are identified by their 0-based index in the list, as seen by `dnd-questions.py --by-entity <slug>`. Indices are unstable across edits — `--answer` reads the list fresh each invocation.
- **Wiki-wide** entries in `questions.md` have an explicit `id:` of the form `q-<tag>-NNN` (e.g. `q-cal-001` for calendar, `q-lor-003` for lore, `q-mec-002` for mechanics). IDs are user-assigned on creation and persist.
- Collision check: `dnd-questions.py --ask` rejects IDs already in use.

## Error Handling

### Lint severity triage (finalized)

| Severity | CLI default | `--strict` | Pre-commit hook | Reports |
|----------|-------------|-----------|-----------------|---------|
| error | exit 1 | exit 1 | blocks | always |
| warning | exit 0 | exit 1 | logs, does not block | always |
| info | exit 0 | exit 0 | logs | verbose only |

Change from v1: pre-commit hook now runs `dnd-lint.py` without `--strict`. Strict stays for manual CI-like runs and for the `6.9` compile-ritual step. This prevents warning spam from blocking legitimate work while keeping errors strictly gated.

### Compile-ritual failure modes

- **Step 6.0 sync-state crash** (missing markers in STATE/CAMPAIGN) → halt; user runs `--migrate`.
- **Step 6.3 rebuild crash** (broken frontmatter YAML) → halt; user fixes the file reported.
- **Step 6.5 verify-compile** never halts — writes to `.cache/verify-<date>.md`, surfaces in step 9 report.
- **Step 6.7 lint --fix** is idempotent; if it cannot auto-apply a fix, the issue persists to 6.9.
- **Step 6.9 lint --strict error** → halt; agent writes partial report, user fixes, user reruns compile (ritual restarts from step 6.0).

### Open-question edge cases

- Agent adds duplicate open_question (same text, same entity) → silent dedupe on `--ask`.
- User answers wrong index → `--answer --dry-run` shows what would happen; `--undo` is out of scope (use git).
- Orphaned open_question (entity deleted) — lint L9 handles (undeclared-fields on the orphan's parent), manually cleaned.

## Testing Approach

### Unit tests (pytest)

Follow existing `tests/test_dnd_lint_*.py` pattern. One test file per new check:

- `test_dnd_lint_bidirectional.py` (L1)
- `test_dnd_lint_body_mentions.py` (L2)
- `test_dnd_lint_session_mentions.py` (L3)
- `test_dnd_lint_rumour_reflected.py` (L4)
- `test_dnd_lint_timeline_parity.py` (L5)
- `test_dnd_lint_item_from_loot.py` (L6)
- `test_dnd_lint_placeholders.py` (L7)
- `test_dnd_lint_stale_questions.py` (L8)
- `test_dnd_lint_undeclared_fields.py` (L9)

Each file: ≥3 test cases (positive/negative/edge).

### Integration tests

- `test_dnd_questions.py` — full lifecycle (ask → list → answer → remove).
- `test_dnd_graph.py` — synthetic wiki → DOT + JSON + `--find-path` correctness.
- `test_dnd_verify_compile.py` — synthetic raw/live + synthetic session → diff output.
- `test_dnd_sync_state_expanded.py` — active_effects and confirmed_rumours blocks.

### Regression

All existing 52 pytest tests must pass unchanged. Any required change to existing tests is a schema-breaking change and goes in the SCHEMA phase commit with a test-update note.

### Manual smoke

After each phase commit, run on the real wiki:
```
python3 scripts/dnd-lint.py --strict
python3 scripts/dnd-sync-state.py --check
python3 scripts/dnd-rebuild.py
python3 scripts/dnd-questions.py
```
Expected: 0 errors, 0 warnings (except documented info like taxonomy-hygiene prefix-containment).

## Roll-out plan (sequential phases, single main branch)

**Note:** All work happens on `main` directly, one phase = one (or a few) commits. No worktree, no branches. Every phase must leave the repo in a `lint --strict` green state (or documented phase skip).

| Phase | Title | Commits | Expected outcome |
|-------|-------|---------|------------------|
| 0 | Spec + plan | `docs: add v2 design spec` | spec + plan committed |
| 1 | SCHEMA expansion | `schema: document all in-use fields + new enums` | SCHEMA covers everything; no data files changed yet |
| 2 | Relations unification | `schema: unify relations format; migrate all entities` | all entities use `relations: [{to, kind}]`; lint green |
| 3 | Data fixes batch | `fix: data integrity — 17 issues` | 7 errors + 10 warnings from audit resolved |
| 4 | Taxonomy cleanup | `schema: reorganize taxonomy (places, mechanics, concepts)` | new taxonomy structure; tags migrated |
| 5 | Open questions infrastructure | `feat: open_questions schema + questions.md + dnd-questions.py + L7 + L8` | ask/list/answer/stale work; subclass TBDs are tracked |
| 6 | Nine new lint checks | Multiple commits, each: `feat(lint): <check-name>` | L1-L9 in place; compile ritual uses L2/L3 in step 6.7 |
| 7 | sync-state expansion | `feat: active_effects + confirmed_rumours derived blocks` | STATE.md auto-reflects party status and confirmed rumours |
| 8 | Items migration | `feat: migrate 8 named items to items/` | items/ populated; loot.md trimmed |
| 9 | New tooling | Multiple commits: `feat: dnd-graph.py`, `feat: dnd-verify-compile.py`, `feat: dnd-search.py faceted`, `feat: dnd-embed.py expanded namespaces` | all new scripts live; embed rebuilt |
| 10 | SKILL.md + README.md updates | `docs: skill contract v2` | process contract matches implementation |
| 11 | Final smoke + embed rebuild | `chore: wiki hardening v2 complete` | everything green; embedding rebuilt; tag-worthy checkpoint |

Each phase commit is preceded by tests (TDD) where applicable. Phases 1-4 are primarily data + schema; phases 5-10 are primarily code; phase 11 is verification.

## Out of scope (explicit)

- `dnd-contradict.py` (embedding-based contradiction mining) — defer to v3.
- Web UI for graph visualization — defer to v3.
- GM-layering — confirmed not needed (single-layer wiki).
- Alternate campaign systems (Pathfinder, 5e2014) — not in scope.
- Rules/SRD lint — existing SRD files are external-reference; no schema applied.

## Resolved design decisions (captured for posterity)

| # | Decision | Chosen | Rationale |
|---|----------|--------|-----------|
| 1 | Scope organization | Single mega-spec, sequential phases | All work must finish before next session; agent-driven review handles context load |
| 2 | Relations format | list-of-dicts `{to, kind}` | Simpler than nested dict; enum `kind` validates; queries via index |
| 3 | GM/party layering | Not implemented — single layer | Every wiki fact is party knowledge; uncertainty via rumour.status + open_questions |
| 4 | Calendar inconsistency (day 47 vs 17) | Open question | Punt to GM; `q-cal-001` in questions.md |
| 5 | `secrets:` field (berser-trant) | Delete field, migrate to body + tag | No undeclared frontmatter fields |
| 6 | Items migration scope | Selective (~8 items + re-eval during work) | Trivial loot stays in loot.md; named items are first-class |
| 7 | Git workflow | main direct, no worktree | User preference |
| 8 | Auto-apply data fixes | Yes, in one batch commit | User grants authority; diff is reviewable |
| 9 | Session mentions backfill | via `dnd-lint.py --fix` (L2/L3) + in step 6.7 of compile ritual | Both paths use same fixer code |
| 10 | Subclass TBDs | `subclass: null` + `subclass_status: tbd` + open_question | Structured, lint-queryable |
| 11 | `dnd-note.sh` dedupe | `--dedupe` flag off by default | Non-breaking; opt-in |
| 12 | Commit convention | `compile(session-N): +X, ~Y, -Z, ?W` for compile; `fix:`, `schema:`, `feat:` otherwise | Parseable prefix |
| 13 | Test coverage | 100% new lint checks + integration for new scripts | Follows existing pattern |

## Success criteria

1. `dnd-lint.py --strict` returns `0 errors, 0 warnings` on the full wiki.
2. `dnd-sync-state.py --check` returns exit 0.
3. Every NPC has `mentions: [session-slug, ...]` populated where applicable.
4. Every session has `mentions` covering all entities named in its body.
5. `items/` contains ≥8 entries; `loot.md` no longer duplicates named items.
6. `questions.md` + at least 3 `open_questions` entries exist (calendar, hp_max, subclass).
7. `dnd-graph.py --orphans` returns empty list (or only documented exemptions like sessions).
8. `dnd-questions.py` successfully runs `--ask` and `--answer` cycle on a test entity.
9. Pre-commit hook blocks on error, allows on warning.
10. All 52 existing tests + new tests pass (`pytest` green).
11. Manual: run a synthetic compile ritual on fresh raw/live, all 11 steps complete without halt.
