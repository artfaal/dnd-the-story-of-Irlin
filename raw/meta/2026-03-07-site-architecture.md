# Архитектура сайта «Летопись Торвина»
*Дата: 2026-03-07*

## Концепция и тон

Сайт — это не нейтральная вики, а **походная летопись от лица Торвина Камнеклятва**:
- голос: сухой, прямой, местами ворчливый;
- фокус: «что важно для выживания, мести и пути в Лускан»;
- структура: одновременно хроника событий + справочник сущностей (люди, места, нити квестов, трофеи, тайны);
- оптика: субъективная (Торвин может ошибаться, сомневаться, язвить).

Редакторский принцип: любой факт в базе либо попадает в публичные разделы, либо уходит в «служебные материалы» (для команды/разработчика), но не теряется.

---

## Разделы сайта

| Раздел (название Торвина) | Hugo section | Источник файлов | Описание |
|---|---|---|---|
| **Книга Дорог** | `chronicle` | `sessions/*.md`, `notes/*.md`, `timeline.md` | Главная хроника: сессии, живые заметки, временная ось. |
| **Нити Клятвы** | `vows` | `quests/*.md`, `quests/_index.md` | Все квесты (главные, активные, замороженные, выполненные). |
| **Лики и Морды** | `faces` | `npcs/*.md`, `npcs/_index.md`, `characters/_party.md`, `characters/*.md` | NPC + партия. Фильтры: союзник/враг/нейтральный, жив/мёртв/неизвестно. |
| **Земли и Переправы** | `roads` | `locations/*.md`, `locations/_index.md`, `assets/maps/*` | Локации, карты, маршруты, телепорт-имена Ленивца. |
| **Кладовая Дружины** | `arsenal` | `loot.md`, релевантные фрагменты из сессий/квестов | Артефакты, кольца, расходники, документы, награды. |
| **Шёпот в Трактире** | `whispers` | `rumours.md` | Слухи с валидацией: неизвестно/подтверждён/опровергнут. |
| **Незакрытые Счёты** | `omens` | выборки из `quests/*`, `sessions/*`, `rumours.md` | Тайны и открытые вопросы (Катастрас, кольца Файрмей, «серебряное касание»). |
| **Устав и Законы Боя** | `codex` | `rules/homebrew.md`, `rules/srd-5e2024*.md` | Игровые правила/справочник механик (в отдельной зоне навигации). |
| **Служебная мастерская** (не в меню игрока) | `workshop` | `CAMPAIGN.md`, `PLAN.md`, `TODO.md`, `report/*.md` | Метаданные кампании, планирование, аналитические отчёты. |

### Маппинг: `dnd/` → Hugo sections

- `dnd/sessions/*.md` → `content/chronicle/sessions/`
- `dnd/notes/*.md` → `content/chronicle/live-notes/`
- `dnd/timeline.md` → `content/chronicle/timeline/_index.md`
- `dnd/quests/*.md` (кроме `_index.md`) → `content/vows/quests/`
- `dnd/quests/_index.md` → `content/vows/_index.md` (или генерируемая сводка)
- `dnd/npcs/*.md` (кроме `_index.md`) → `content/faces/npcs/`
- `dnd/characters/*.md` → `content/faces/party/`
- `dnd/characters/_party.md` → `content/faces/party/_index.md`
- `dnd/npcs/_index.md` → `content/faces/npcs/_index.md`
- `dnd/locations/*.md` (кроме `_index.md`) → `content/roads/places/`
- `dnd/locations/_index.md` → `content/roads/_index.md`
- `dnd/loot.md` → `content/arsenal/_index.md`
- `dnd/rumours.md` → `content/whispers/_index.md`
- `dnd/rules/*.md` → `content/codex/`
- `dnd/CAMPAIGN.md` → `content/workshop/campaign/_index.md`
- `dnd/PLAN.md` → `content/workshop/plan/_index.md`
- `dnd/TODO.md` → `content/workshop/todo/_index.md`
- `dnd/report/*.md` → `content/workshop/reports/`
- `dnd/assets/**` → `static/assets/**` (без трансформаций путей)
- `dnd/vectors.db` → **не публиковать** (исключить из контента/статики)

---

## Главная страница

Главная (`/`) — «лист дежурного Торвина», собирается из блоков:

1. **Последняя запись в хронике** (последняя сессия + 5 ключевых событий).
2. **Активные нити клятвы** (топ 5 активных квестов, отдельно главный квест «Путь в Лускан»).
3. **Срочные хвосты к следующей сессии** (ручной блок из frontmatter `next_session_focus`).
4. **Разыскиваются ответы** (3–6 открытых загадок из `omens`).
5. **Карта текущего театра** (актуальная карта региона/локации, например `northern-keller-region.jpg`).
6. **Состояние партии** (кто в строю, кратко по персонажам).
7. **Фраза дня Торвина** (авто-цитата из последней сессии, fallback: «ЗДЕСЬ БЫЛА БИТВА»).

---

## Hugo тема — выбор и обоснование

**Рекомендация: `hugo-book` как базовая тема + кастомный скин «Летопись Торвина».**

Почему:
- отлично подходит под иерархический контент (разделы/подразделы как в кампании);
- встроенный поиск (Lunr/FlexSearch), хорошая sidebar-навигация;
- хорошо переваривает большие markdown-файлы;
- легко кастомизируется под тёмный «пергамент+бронза» стиль.

Альтернатива: `Relearn` (если нужен более «доковый» UX и расширенные shortcodes). Но для минимального риска и скорости старта — `hugo-book`.

---

## Навигация и связи между страницами

### 1) Глобальная навигация
- Левая панель: 8 разделов летописи (без `workshop` в публичном меню).
- Верхняя строка: быстрые фильтры `Квесты`, `NPC`, `Локации`, `Сессии`, `Тайны`, `Поиск`.

### 2) Межстраничные связи (граф)
В frontmatter каждой сущности использовать slug-ссылки:
- `related_npcs: [rouen, irvin-dandrich]`
- `related_locations: [northern-keller, temple-ilmater]`
- `related_quests: [main-luskan, identify-rings]`
- `related_sessions: [2026-03-07]`

В шаблонах single-страниц выводить блоки:
- «Связанные лица»
- «Связанные места»
- «Связанные нити»
- «Где это всплывало в хронике»

### 3) Таксономии
- `tags` — свободные теги
- `status` — статус квестов/сущностей
- `relation` — союзник/враг/нейтральный
- `region` — региональная привязка

### 4) Shortcodes
Нужные shortcodes:
- `{{< npc "slug" >}}` — карточка/ссылка NPC
- `{{< quest "slug" >}}` — карточка квеста
- `{{< place "slug" >}}` — карточка локации
- `{{< map src="..." caption="..." >}}` — карта с подписью
- `{{< clue text="..." status="unknown|confirmed|false" >}}` — маркер тайны/улики

---

## Frontmatter — что добавить в dnd-файлы

Минимальный унифицированный набор для всех типов:

```yaml
---
title: "Человекочитаемое название"
slug: "stable-slug"
type: "session|quest|npc|location|character|rumour|loot|rule|meta"
date: 2026-03-07
lastmod: 2026-03-07
draft: false
weight: 0
tags: []
series: []
status: "active|done|frozen|alive|dead|unknown|explored|unexplored"
summary: "Короткая выжимка в 1-2 строки"
torvin_voice: "dry|grumpy|proud|neutral"
related_npcs: []
related_locations: []
related_quests: []
related_sessions: []
assets: []
search_boost: 1
---
```

Дополнения по типам:
- **session**: `number`, `duration_hours`, `location_ingame`, `next_session_focus`
- **quest**: `priority`, `giver`, `reward`, `deadline`
- **npc/character**: `race`, `role`, `relation`, `first_seen`, `portrait`
- **location**: `type_loc`, `region`, `teleport_name`, `map`
- **rumour/clue**: `source`, `verification_status`, `confidence`

---

## ТЗ для разработчика

### Структура Hugo проекта

```text
/Users/claw/.openclaw/workspace/dnd-site/
  hugo.toml
  go.mod
  themes/
    hugo-book/                # git submodule или pinned copy
  content/
    chronicle/
      _index.md
      sessions/
      live-notes/
      timeline/
    vows/
      _index.md
      quests/
    faces/
      _index.md
      npcs/
      party/
    roads/
      _index.md
      places/
    arsenal/
      _index.md
    whispers/
      _index.md
    omens/
      _index.md
    codex/
      _index.md
    workshop/
      _index.md
      campaign/
      plan/
      todo/
      reports/
  layouts/
    _default/
    partials/
    shortcodes/
  assets/
    scss/
      main.scss
    fonts/
  static/
    assets/                   # symlink или rsync-копия из dnd/assets
  scripts/
    sync-dnd-content.sh
    build.sh
  Dockerfile
  docker-compose.yml
```

### Конфиг `hugo.toml`

Обязательные блоки:
- `baseURL`, `languageCode = "ru"`, `defaultContentLanguage = "ru"`
- `theme = "hugo-book"`
- `enableRobotsTXT = true`
- `markup.goldmark.renderer.unsafe = true` (если нужны rich-вставки)
- `taxonomies`: `tag`, `status`, `relation`, `region`, `arc`
- `outputs.home = ["HTML", "RSS", "JSON"]` (JSON для поиска)
- `params`: бренд «Летопись Торвина», логотип/иконка, тёмная палитра
- меню разделов в стиле летописи (названия Торвина, не технические)

### Скрипт синхронизации контента

Рекомендация: **rsync + нормализация frontmatter**, не прямой symlink на весь `dnd/`.

`sync-dnd-content.sh` делает:
1. Очищает staging-каталог `content/` (кроме hand-written `_index.md` при необходимости).
2. Копирует `.md` по маппингу разделов.
3. Копирует `dnd/assets/**` в `static/assets/**`.
4. Для файлов без `title`/`date` добавляет/нормализует frontmatter.
5. Генерирует агрегаты:
   - `content/omens/_index.md` (открытые вопросы);
   - `content/chronicle/timeline/_index.md` (из `timeline.md`);
   - `content/arsenal/_index.md` (из `loot.md`).
6. Валидирует битые ссылки (`hugo --printPathWarnings`).

### Dockerfile

Цель: production build + лёгкий runtime.

- Stage 1 (`klakegg/hugo:ext` или `gohugoio/hugo`):
  - копия проекта;
  - `hugo --minify`;
- Stage 2 (`nginx:alpine`):
  - копия `public/` в `/usr/share/nginx/html`;
  - кастомный `nginx.conf` (gzip, cache headers для `assets/*`);
  - expose порта контейнера `80`.

### Deploy-скрипт (`/Users/claw/.openclaw/workspace/scripts/dnd-deploy.sh`)

Что должен делать пошагово:
1. `set -euo pipefail`.
2. Проверка входных директорий (`/Users/claw/.openclaw/workspace/dnd`, `.../dnd-site`).
3. Запуск `scripts/sync-dnd-content.sh`.
4. Локальная валидация: `hugo --minify --gc`.
5. `docker build -t dnd-chronicle:latest .`.
6. Остановка старого контейнера (если есть): `docker rm -f dnd-chronicle || true`.
7. Запуск нового контейнера:
   - `-p 127.0.0.1:8765:80` (по умолчанию);
   - `--restart unless-stopped`.
8. Smoke-check: `curl -f http://127.0.0.1:8765/`.
9. (Опционально) reload nginx reverse proxy на `pihome.local`, если используется внешний маршрут.
10. Лог результата (дата, git commit, статус) в `dnd-site/deploy.log`.

### Деплой на `pihome.local`

Вариант по умолчанию:
- контейнер слушает `127.0.0.1:8765`;
- Nginx на хосте проксирует `http://pihome.local/dnd/` → `127.0.0.1:8765`;
- наружу в интернет не публикуется.

Fallback:
- без nginx, прямой доступ `http://pihome.local:8765`.

### Стилизация

Палитра:
- фон: `#1B1A18`
- основной текст: `#D9C8A9`
- акцент/бронза: `#A87832`
- ссылки: `#4E6E6A`
- опасность/враг: `#8F3A2E`

Типографика:
- заголовки: Cinzel / Marcellus SC;
- основной текст: Inter / Noto Sans;
- цитаты Торвина: полукурсив, рамка как полевая заметка.

UI-элементы:
- иконки разделов (lucide/heroicons): меч, свиток, шлем, карта, мешок, глаз, молот.
- бейджи статуса (`активный`, `выполнен`, `мёртв`, `неизвестно`) цветокодом.
- блок «слух/факт» с разными визуальными маркерами.

---

## Технические решения по контенту

1. **Источник истины** — `/Users/claw/.openclaw/workspace/dnd/`.
2. Hugo-проект — отдельная папка `dnd-site`, чтобы не смешивать редакторскую базу и генерацию.
3. Синхронизация контента — скриптом (повторяемо, проверяемо, без ручного копипаста).
4. `rules/srd-5e2024*.md` оставлять в отдельной зоне (`codex`) и исключать из главной поисковой выдачи хроники (иначе «зашумит» сюжет).
5. `vectors.db` не публиковать и не монтировать в web-root.

---

## Критерии приёмки реализации (для Кремня)

- Все файлы из `dnd/` учтены маппингом (публичный раздел или служебный раздел, либо исключение с обоснованием).
- На главной есть: последняя сессия, активные квесты, открытые тайны, карта, статус партии.
- Работает поиск по контенту Hugo.
- На каждой сущности отображаются связанные сущности.
- Тёмная «летописная» тема применена и читабельна.
- Контейнер на `pihome.local` поднимается и отдаёт сайт стабильно.
- Deploy-скрипт выполняет полный цикл sync → build → run → smoke-check.
