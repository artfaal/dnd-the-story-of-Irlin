# D&D Campaign Wiki — справка

Живая вики кампании **«The story of Irlin»**. Набор markdown-файлов со структурированным frontmatter, управляется AI-скиллом `dnd`.

Данные лежат в этой папке (`~/PROJECTS/dnd-wiki/`). Скилл живёт отдельно (`~/.openclaw/skills/dnd/`, симлинки в `~/.claude/skills/dnd` и `~/.hermes/skills/gaming/dnd`) — один и тот же код работает из Claude Code, openclaw, hermes.

---

## TL;DR — типичная игровая сессия

```bash
# 1. Перед сессией — краткий бриф
DND_DIR=~/PROJECTS/dnd-wiki python3 ~/.openclaw/skills/dnd/scripts/dnd-summary.py

# 2. Во время сессии — AI сам создаёт raw/live/YYYY-MM-DD.md при первой фразе «запиши …»
#    или вручную:
~/.openclaw/skills/dnd/scripts/dnd-note.sh --npc "Имя — роль"

# 3. После сессии — одно слово AI:
#    «структурируй»
# скилл сам запустит 9-шаговый compile ritual и закоммитит изменения.
```

---

## Как обращаться к AI

Скилл `dnd` триггерится из любой харнесс-сессии (Claude Code / openclaw / hermes). Не нужно вызывать его явно — достаточно ключевых фраз.

| Что сказать                                           | Что произойдёт                                    |
|------------------------------------------------------|---------------------------------------------------|
| «саммари / что было / напомни / перед сессией»       | запустит `dnd-summary.py`, выведет бриф           |
| «запиши …»  /  «нпс: Имя — роль»  /  «квест: …»      | добавит строку в `raw/live/YYYY-MM-DD.md`         |
| «слух: …»   /  «лут: …»                              | то же, с тегом                                    |
| «структурируй» (после сессии)                        | compile ritual — создаст `sessions/YYYY-MM-DD.md`, обновит NPCs/quests/STATE/index, залинтит, коммит |
| «найди / кто такой / что мы знаем про …»             | семантический поиск (`dnd-search.py`)             |
| «обнови индексы»                                     | `dnd-rebuild.py` → `index.md`                     |
| «переиндексируй эмбеддинги»                          | `dnd-embed.py` → `vectors.db`                     |
| «валидация / lint»                                   | `dnd-lint.py` + отчёт                             |
| «новая кампания в ~/PROJECTS/foo»                    | `dnd-init.py --path ~/PROJECTS/foo`               |

AI следует **read-first** правилу: сначала читает `SCHEMA.md` + `STATE.md` + `index.md` + хвост `log.md`, потом что-то делает.

---

## Жизненный цикл одной сессии

### Pre-session (за 5-15 минут до игры)

Скажи: **«саммари»** (или запусти вручную `dnd-summary.py`). Получаешь:
- дату и номер предыдущей сессии + её one-liner
- текущую локацию, уровни партии, активные эффекты (из `STATE.md`)
- активные квесты по приоритету (🔴 главные → ❗ срочные → 🟡 основные → ⚪ побочные)
- блок «С прошлой сессии» — что изменилось по `log.md`

Флаги:
```bash
dnd-summary.py --last 2        # две последние сессии
dnd-summary.py --quests-only   # только квесты
dnd-summary.py --full          # включая побочные
```

### Live (во время игры)

AI при первой фразе «запиши / нпс: / квест: / лут: / слух:» **автоматически создаёт** файл `raw/live/YYYY-MM-DD.md` и дописывает туда timestamped строки. Файл — страховка: если контекст сессии сбросится, данные сохранятся на диске.

Ты можешь также делать заметки вручную через shell:
```bash
~/.openclaw/skills/dnd/scripts/dnd-note.sh "свободная заметка"
~/.openclaw/skills/dnd/scripts/dnd-note.sh --npc   "Имя — роль"
~/.openclaw/skills/dnd/scripts/dnd-note.sh --quest "Название — суть"
~/.openclaw/skills/dnd/scripts/dnd-note.sh --rumour "текст слуха"
~/.openclaw/skills/dnd/scripts/dnd-note.sh --loot  "предмет"
# если сессия играется задним числом:
~/.openclaw/skills/dnd/scripts/dnd-note.sh --session 2026-05-01 "…"
```

### Post-session

Когда сессия закончилась — скажи AI **«структурируй»**. Он прогонит **9-шаговый compile ritual**:

1. Читает `raw/live/YYYY-MM-DD.md` + SCHEMA + STATE + index + хвост log.
2. Парсит события по категориям: NPC / quest / location / loot / rumour / state-change / decision / mystery.
3. Решает для каждого: update существующей страницы / create новую (если проходит порог) / только в session log.
4. Пишет `sessions/YYYY-MM-DD.md` с frontmatter и структурными разделами.
5. Обновляет `STATE.md`, `timeline.md`, `rumours.md`, `loot.md`.
6. Запускает `dnd-rebuild.py` (index) и `dnd-lint.py` (валидация). Опционально `dnd-embed.py`.
7. Append в `log.md`: компактная сводка что создано/обновлено/закрыто.
8. `git add . && git commit` с сообщением `compile: session YYYY-MM-DD (N updates)`.
9. Отчёт пользователю: сводка + warnings + commit hash.

**Пороги создания новой страницы** (из SCHEMA):
- **NPC** — упомянут по имени 2+ раз ИЛИ центральный в одной сессии.
- **Quest** — партия взяла цель (есть `giver` + task).
- **Location** — посетили ИЛИ описана детально 2+ раз.
- **Session** — всегда, одна на сессию.
- **Character** — всегда, одна на PC.
- **Item** — магический / именной / сюжетный. Обычный лут идёт в `loot.md`.
- **Rumour** — YAML-запись в `rumours.md`. Отдельный quest-файл только когда партия действует по слуху.

---

## Структура данных

```
~/PROJECTS/dnd-wiki/
├─ README.md           ← этот файл
├─ SCHEMA.md           ← контракт: типы, enum'ы, taxonomy, required поля
├─ CAMPAIGN.md         ← название кампании, игроки, лор-константы (меняется редко)
├─ STATE.md            ← живое состояние: уровни, локация, эффекты, next_session
├─ index.md            ← автогенерация rebuild'ом, не править руками
├─ log.md              ← append-only audit trail всех скриптов
├─ rumours.md          ← YAML-слухи с статусом
├─ loot.md             ← накопленный обычный лут
├─ timeline.md         ← хронология событий кампании
│
├─ npcs/               ← по файлу на NPC:           <slug>.md
├─ quests/             ← по файлу на квест:         <slug>.md
├─ locations/          ← по файлу на локацию:       <slug>.md
├─ sessions/           ← сессии:                    YYYY-MM-DD.md
├─ characters/         ← PC персонажи игроков:      <slug>.md (+ _party.md)
├─ items/              ← магические/именные предметы
├─ rules/              ← хомрулы + SRD 5e 2024 (EN/RU)
│
├─ raw/
│  ├─ live/            ← live-notes сессий (raw/live/YYYY-MM-DD.md)
│  ├─ handouts/        ← хэндауты, документы от ДМ
│  ├─ maps/            ← карты
│  ├─ photos/          ← фото-артефакты
│  └─ meta/            ← служебное (lint baselines, todo-списки)
│
├─ assets/             ← портреты, карты, изображения (для markdown-ссылок)
└─ vectors.db          ← SQLite с embedding'ами (bge-m3 от Ollama; gitignore'нут)
```

**Naming conventions:**
- Имена файлов: `lowercase-hyphenated.md`, ASCII latin (кириллица транслитерируется).
- `slug:` во frontmatter == filename без `.md` (primary key).
- `name:` — может быть кириллицей, с пробелами, любой читабельный заголовок.
- Sessions: `sessions/YYYY-MM-DD.md`. Live notes: `raw/live/YYYY-MM-DD.md`.

---

## Frontmatter cheatsheet

Обязательные поля **для любой** страницы: `name`, `slug`, `type`, `created`, `updated`, `tags`, `sources`.

Плюс **type-specific** required:

| Тип        | Папка        | Type-specific required                               |
|------------|-------------|------------------------------------------------------|
| npc        | `npcs/`      | `race`, `role`, `location`, `status`, `relation`     |
| quest      | `quests/`    | `status`, `priority`, `giver`                        |
| location   | `locations/` | `type_loc`, `region`, `status`                       |
| session    | `sessions/`  | `number`, `date`, `summary_one_line`                 |
| character  | `characters/`| `player`, `race`, `classes`, `level`, `status`       |
| item       | `items/`     | `owner`, `origin`, `rarity`                          |
| rule       | `rules/`     | `kind` (`homebrew` или `raw`)                        |
| rumour     | в `rumours.md` | — (yaml-запись, не отдельный файл)                 |

**Enum'ы** (полный список — в `SCHEMA.md`, секция `## Enums`):
- npc.status: `живой / мёртвый / пропал / неизвестно`
- npc.relation: `союзник / нейтральный / подозрительный / враг / неизвестно`
- quest.status: `активный / выполнен / провален / заморожен`
- quest.priority: `главный / основной / побочный / срочно`
- character.status: `healthy / wounded / cursed / dying / deceased`
- …и т.д.

**Cross-references**: только через YAML (`relations:`, `mentions:`, `sources:`, `giver:`, `location:`). Никаких `[[wikilinks]]` в теле — тело это narrative markdown.

**Update policy**:
1. Новая инфа конфликтует со старой — более свежий источник обычно побеждает, `updated:` бамп.
2. Если противоречие принципиальное — оставить обе версии, `### Противоречие [YYYY-MM-DD]` блок в теле + `contradictions: [<slug>]` во frontmatter.
3. `updated:` всегда пересчитывается при правке.

---

## CLI reference

Все скрипты самолокализуются и используют `DND_DIR` (default `$HOME/PROJECTS/dnd-wiki`). Достаточно запустить по абсолютному пути, или добавить `~/.openclaw/skills/dnd/scripts` в `PATH`.

### `dnd-summary.py` — pre-session бриф
```bash
python3 ~/.openclaw/skills/dnd/scripts/dnd-summary.py [--last N] [--quests-only] [--full]
```

### `dnd-note.sh` — live-заметка
```bash
dnd-note.sh [--npc|--quest|--rumour|--loot] "текст"
dnd-note.sh --session YYYY-MM-DD "текст"  # писать в чужой день
```
Пишет в `raw/live/<today>.md` (создавая файл с заголовком, если ещё нет) + лог-запись.

### `dnd-rebuild.py` — перестроить index.md
```bash
python3 ~/.openclaw/skills/dnd/scripts/dnd-rebuild.py
```
Генерирует `index.md` из всех папок типов. Quests группирует по `priority`. `updated: 2026-MM-DD | Total: N pages`.

### `dnd-lint.py` — валидация SCHEMA
```bash
python3 ~/.openclaw/skills/dnd/scripts/dnd-lint.py
python3 ~/.openclaw/skills/dnd/scripts/dnd-lint.py --strict    # warnings → errors
python3 ~/.openclaw/skills/dnd/scripts/dnd-lint.py --fix       # авто-фикс поддающихся (missing-index, missing `updated`)
python3 ~/.openclaw/skills/dnd/scripts/dnd-lint.py --only missing-required   # один чек
python3 ~/.openclaw/skills/dnd/scripts/dnd-lint.py --format json             # машиночитаемо
```
**9 проверок**: `missing-required`, `invalid-type`, `invalid-enum`, `broken-ref`, `missing-index`, `orphan`, `taxonomy`, `temporal`, `unique-slug`.

### `dnd-search.py` — семантический поиск
```bash
python3 ~/.openclaw/skills/dnd/scripts/dnd-search.py "кольца семьи Файрмей"
python3 ~/.openclaw/skills/dnd/scripts/dnd-search.py --namespace rules "grapple"
python3 ~/.openclaw/skills/dnd/scripts/dnd-search.py --top 20 --verbose "query"
```
Требует `vectors.db`. Если не построен — сперва `dnd-embed.py`.

### `dnd-embed.py` — построить/обновить vectors.db
```bash
python3 ~/.openclaw/skills/dnd/scripts/dnd-embed.py                       # инкремент для all
python3 ~/.openclaw/skills/dnd/scripts/dnd-embed.py --namespace campaign  # только кампания
python3 ~/.openclaw/skills/dnd/scripts/dnd-embed.py --namespace rules     # только правила
python3 ~/.openclaw/skills/dnd/scripts/dnd-embed.py --force               # полная пересборка
python3 ~/.openclaw/skills/dnd/scripts/dnd-embed.py --dry-run             # посмотреть что будет
```

### `dnd-init.py` — бутстрап новой кампании
```bash
python3 ~/.openclaw/skills/dnd/scripts/dnd-init.py --path ~/PROJECTS/new-game \
    --campaign "Название" --system "D&D 5e 2024" --dm "TBD"
python3 ~/.openclaw/skills/dnd/scripts/dnd-init.py --non-interactive    # без вопросов
```
Создаёт структуру папок, SCHEMA + CAMPAIGN + STATE + index + log skeletons, `git init`, `.gitignore`.

---

## Как добавить новое вручную (без AI)

Минимальный пример NPC `npcs/new-npc.md`:

```yaml
---
name: Имя Персонажа
slug: new-npc
type: npc
race: Человек
role: Кузнец в Северном Келлере
location: northern-keller
status: живой
relation: нейтральный
first_seen: '2026-04-24'
created: '2026-04-24'
updated: '2026-04-24'
tags: [северный-келлер, кузнец]
sources: []
---

## Что известно
…

## Связи
…
```

После — `dnd-rebuild.py` (index) + `dnd-lint.py` (валидация). Если упадёт — посмотри что именно не сходится.

`tags:` берутся **только** из SCHEMA.md taxonomy. Если нужен новый тег — добавь его в соответствующий список (`regions`, `npcs-named`, `items`, …).

---

## Environment

```bash
# куда писать / откуда читать
export DND_DIR=~/PROJECTS/dnd-wiki           # default — можно не ставить

# ollama для search/embeddings
export DND_OLLAMA_URL=http://forge.lan:11434 # default
export DND_EMBED_MODEL=bge-m3                # default

# скрипты принудительно игнорируют HTTP(S)_PROXY для ollama-вызовов
```

---

## Multi-campaign

Одна кампания = одна папка. Под вторую:

```bash
DND_DIR=~/PROJECTS/second-campaign python3 ~/.openclaw/skills/dnd/scripts/dnd-init.py --path ~/PROJECTS/second-campaign
```

Переключаешься — просто меняешь `DND_DIR` в шелле:
```bash
DND_DIR=~/PROJECTS/second-campaign dnd-summary.py
```

---

## Troubleshooting

### `dnd-lint.py` показывает ошибки
- `missing-required` — не хватает обязательных полей. См. cheatsheet выше или `SCHEMA.md`.
- `invalid-enum` — значение не из enum'а. Подправь на допустимое (перечислены в SCHEMA → Enums).
- `broken-ref` — ссылка на slug, которого нет. Либо опечатка, либо страница не создана.
- `missing-index` — страница не попала в `index.md`. Запусти `dnd-rebuild.py`.
- `orphan` — на страницу никто не ссылается. Добавь её в `mentions:` соответствующей сессии или в `relations:` другой страницы.
- `taxonomy` — тег не в SCHEMA.md taxonomy. Расширь таксономию или убери тег.
- `temporal` — дата `updated` раньше `created`. Проверь.
- `unique-slug` — два файла с одинаковым `slug:`. Переименуй один.

`dnd-lint.py --fix` автоматически решает `missing-index` (запустит rebuild) и некоторые `updated`-пропуски.

### `dnd-search.py` падает или ничего не находит
Vectors.db не построен или устарел. Перестрой: `dnd-embed.py` (инкремент) или `dnd-embed.py --force` (полностью).
Ollama доступен? `curl http://forge.lan:11434/api/tags` — должен вернуть список моделей.

### Скилл не триггерится из AI
Проверь что скилл виден: в Claude Code набери `/` и ищи `dnd` в списке. Если нет — симлинк слетел.
```bash
ls -la ~/.claude/skills/dnd
# должен указывать на ~/.openclaw/skills/dnd
```
Если сломан — пересоздай:
```bash
ln -snf ~/.openclaw/skills/dnd ~/.claude/skills/dnd
```

### Данные пропали
Всё под git. `git log` в `~/PROJECTS/dnd-wiki/` покажет историю.  `git reset --hard <commit>` — откат (осторожно).

---

## Архитектура коротко

- **Data** (эта папка) — markdown + git. Переносимо: склонировал репу на новом хосте → скилл работает.
- **Skill code** (`~/.openclaw/skills/dnd/`) — Python scripts + shared `lib/kb_common.py`. Всё через `DND_DIR`, нет хардкода путей.
- **Tests** (в исходниках скилла, не в deployed) — 52 pytest на `lib` + integration на `scripts`.
- **AI contract** — `SKILL.md` в корне скилла. Определяет триггер-фразы, 9-шаговый compile ritual, и правило read-first.

Изменения в коде скилла — только через worktree + тесты + redeploy.
