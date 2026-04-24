# Schema — D&D Campaign Wiki

This file defines the contract for the campaign wiki: entity types, required frontmatter fields, taxonomy, enums, update policy, naming conventions.

Linter (`dnd-lint.py`) parses the machine-readable `<!-- schema:... -->` blocks and enforces them. The prose sections document intent for humans.

## Domain

A single D&D campaign tracked as interlinked markdown files. Campaign-specific identity lives in CAMPAIGN.md. This schema file is domain-agnostic — it works for any D&D 5e campaign. Swap out taxonomy values to fit your campaign.

## Types

Every page has `type:` in frontmatter. Valid types are listed below.

<!-- schema:types -->
- npc
- quest
- location
- session
- character
- item
- rule
- rumour
<!-- /schema:types -->

## Enums

### npc

<!-- schema:enum:npc:status -->
- живой
- мёртвый
- пропал
- неизвестно
<!-- /schema:enum:npc:status -->

<!-- schema:enum:npc:relation -->
- союзник
- нейтральный
- подозрительный
- враг
- неизвестно
<!-- /schema:enum:npc:relation -->

### quest

<!-- schema:enum:quest:status -->
- активный
- выполнен
- провален
- заморожен
<!-- /schema:enum:quest:status -->

<!-- schema:enum:quest:priority -->
- главный
- основной
- побочный
- срочно
<!-- /schema:enum:quest:priority -->

### location

<!-- schema:enum:location:status -->
- исследована
- частично
- слышали
- не были
<!-- /schema:enum:location:status -->

<!-- schema:enum:location:type_loc -->
- город
- храм
- подземелье
- регион
- корабль
- таверна
- ориентир
<!-- /schema:enum:location:type_loc -->

### character

<!-- schema:enum:character:status -->
- healthy
- wounded
- cursed
- dying
- deceased
<!-- /schema:enum:character:status -->

<!-- schema:enum:character:subclass_status -->
- chosen
- tbd
<!-- /schema:enum:character:subclass_status -->

### item

<!-- schema:enum:item:rarity -->
- common
- uncommon
- rare
- very-rare
- legendary
- artifact
<!-- /schema:enum:item:rarity -->

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

### rule

<!-- schema:enum:rule:kind -->
- homebrew
- raw
<!-- /schema:enum:rule:kind -->

### rumour

<!-- schema:enum:rumour:status -->
- неизвестно
- подтверждён
- опровергнут
- в работе
<!-- /schema:enum:rumour:status -->

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

## Taxonomy

Tags must come from this taxonomy. Add new tags here before using them.

<!-- schema:taxonomy -->
regions: [лускан, побережье-меча, побережье, острова-ирлин, остров, остров-болото, материк, лес, ирлинский-лес, природа, водопад, болота]
cities: [северный-келлер, южный-келлер, храм-ильматера, город, город-угасания, академия-магии, огненные-ладони, арена, дом-прессы, библиотека, чайная, таверна, голодный-лесник, горный-лесник, ферма-самблео, заброшенная-лесопилка, кемп-орков, склеп, катакомбы, подземелье, руины, алтарь, лаборатория]
factions: [академия-огненные-ладони, братство-взирающих-очей, орден-каменной-клятвы, круг-луны, семья-файрмей, братья-бись, группа-приключенцев, первая-группа]
races: [дворф, эльф, человек, тифлинг, полурослик, дракон-рождённый]
classes: [паладин, друид, маг, колдун, плут, клерик, воин, варвар]
status-markers: [активный, заморожен, проклятие, срочно, главный-квест, побочный, выполнен, мёртв, погиб, квест, основная-локация, цель, враг]
mechanics: [mummy-rot, договор-души, обелиски, жаровни, ментальная-атака, электричество, перемещение, механика, демиплан, выход-с-острова, крушение, самовзрыв, побег, прорицание, карта, зарисовка, опознание, ваншот, апгрейд, клятва-мести, мистерия]
creatures: [призрак, призраки, нежить, мумия, гомункул, кракен, мимик, питомец, шершни, шершень-символ, орки, гоблины]
items: [амулет, кольцо, волынка, ячейки, книга-заклинаний, камень, камень-с-шершнем, записка, письмо, письмо-женевьевы, воздушный-шар, гейзенберг, алигары, кресло, повозка]
npcs-named: [роуан-файрмей, бабушка-любава, гидеон, мануэль, лиана, миала, эймар, маргарита, катрин, торвин, ашен, женевьева, самблео, линда-чит, лала-эншейн, берсер-трант, алистер, ильматер]
roles: [профессор, капитан, журналист, мэр, трактирщик, проводник, жрица, пасечник, прорицательница, торговец-информацией, исследователь, защитник, ученик, подозреваемый, подозрительная, беременная, любопытная, важный, добродетель, антагонист, загадочный-маг, жертва, торговцы, власть]
events: [катастрас, лабиринт, скандал, статья, слух, выставка, аукцион, выкуп, путешествие, проход, огненный-круг, корабль, торговля, все-локации]
<!-- /schema:taxonomy -->

## Required fields per type

All types require the common frontmatter: `name`, `slug`, `type`, `created`, `updated`, `tags`, `sources`. Type-specific requirements:

<!-- schema:required:npc -->
- race
- role
- location
- status
- relation
<!-- /schema:required:npc -->

<!-- schema:required:quest -->
- status
- priority
- giver
<!-- /schema:required:quest -->

<!-- schema:required:location -->
- type_loc
- region
- status
<!-- /schema:required:location -->

<!-- schema:required:session -->
- number
- date
- summary_one_line
<!-- /schema:required:session -->

<!-- schema:required:character -->
- player
- race
- classes
- level
- status
<!-- /schema:required:character -->

<!-- schema:required:item -->
- owner
- origin
- rarity
<!-- /schema:required:item -->

<!-- schema:required:rule -->
- kind
<!-- /schema:required:rule -->

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
- secrets
<!-- /schema:optional:npc -->

> Note: `secrets` is a deprecated migration bridge — will be removed in Phase 3 (data fixes). New NPC files must not use it; use body sections + tags for hidden-facts content.

<!-- schema:optional:quest -->
- next_session
- deadline
- blocker
- related_npcs
- related_locations
- created_session
- created_date
- location
- completed_date
- completed_session
- found
<!-- /schema:optional:quest -->

<!-- schema:optional:location -->
- related_npcs
- visited
- map
- parent_region
- sub_locations
- teleport_name
<!-- /schema:optional:location -->

<!-- schema:optional:session -->
- players
- absent
- game_day
- location_ingame
- mentions
- tags
- duration_hours
<!-- /schema:optional:session -->

<!-- schema:optional:character -->
- hp_max
- subclass
- subclass_status
- age
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

## Page thresholds

- **npc:** create when mentioned by name 2+ times OR central to one session.
- **quest:** create when the party commits to a goal (giver + task).
- **location:** create when visited OR described with specific details 2+ times.
- **session:** one page per session (always, at compile).
- **character:** one per party PC (always).
- **item:** create if magical / named / quest-relevant. Mundane loot goes to `loot.md`.
- **rule:** create per distinct homebrew rule OR referenced RAW section.
- **rumour:** yaml record in `rumours.md`. Split into its own quest file only when party acts on it.

## Relations policy (YAML-only)

All cross-references live in frontmatter `relations:` and `mentions:` — never `[[wikilinks]]` in body. Body text is narrative markdown.

- `relations:` — structured map (varies by type).
- `mentions:` — list of session slugs where this page appears.
- `sources:` — where the initial info came from.

Linter checks all slugs in these fields point to existing pages.

## Update policy

When new info conflicts with existing:
1. Check dates — newer source generally wins.
2. If genuinely contradictory, keep both. Add `### Противоречие [YYYY-MM-DD]` block in body; mark in frontmatter `contradictions: [other-slug]`.
3. Bump `updated:` on each edit.

## Naming conventions

- Filenames: `lowercase-hyphenated.md`, ASCII latin (transliterate Cyrillic).
- `slug:` = filename without `.md`. Primary key.
- `name:` can be Cyrillic / contain spaces / punctuation.
- Session files: `sessions/YYYY-MM-DD.md`.
- Live notes: `raw/live/YYYY-MM-DD.md`.

## Open questions policy

Entity frontmatter may carry an `open_questions:` list. Wiki-wide questions live in `questions.md`.

Entry schema:

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
