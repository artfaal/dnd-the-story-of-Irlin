# Action Log — The story of Irlin

> Append-only. Format: `## [YYYY-MM-DD HH:MM] <action> | <subject>`
> Actions: init | ingest | compile | update | create | archive | lint | query | note | migrate | rebuild | embed
> Rotate at >500 entries → rename log-YYYY.md, start fresh.

## [2026-04-24 00:00] migrate | Phase 1 complete
- Source: /Users/claw/.openclaw.pre-reinstall-20260414-150346/workspace/dnd
- Target: ~/PROJECTS/dnd-wiki
- Moved: notes/ → raw/live/
- Removed: npcs/_index.md, quests/_index.md, locations/_index.md
- Moved: PLAN.md, TODO.md → raw/meta/
- Created: SCHEMA.md (skeleton), STATE.md, log.md, index.md (skeleton), items/, raw/{handouts,maps,photos,meta}/
- Written: CAMPAIGN.md (extracted from legacy SKILL.md)

## [2026-04-24 01:00] migrate | Phase 1 fixup
- Moved: assets/npcs/*.jpg → raw/photos/ (NPC portraits)
- Moved: assets/item-musical-instrument-1984.jpg → raw/photos/
- Moved: report/*.md → raw/meta/ (legacy analysis docs)
- Removed: empty assets/ and report/ directories

## [2026-04-24 10:17] rebuild | index.md
- NPCs: 33
- Quests: 22
- Locations: 7
- Sessions: 5
- Characters: 5
- Items: 0
- Rules: 0
- Total: 72

## [2026-04-24 10:25] embed | namespace: all
- new: 1
- skipped: 78
- updated: 0

## [2026-04-24 10:26] summary | pre-session
- last=1
- full=False

## [2026-04-24 10:36] lint | 371 errors, 291 warnings
- check missing-required
- check invalid-type
- check invalid-enum
- check broken-ref
- check missing-index
- check orphan
- check taxonomy
- check temporal
- check unique-slug

## [2026-04-24 10:42] summary | pre-session
- last=1
- full=False

## [2026-04-24 10:42] summary | pre-session
- last=1
- full=False

## [2026-04-24 10:42] note | plain Phase 2 verification note

## [2026-04-24 10:42] summary | pre-session
- last=1
- full=False

## [2026-04-24 10:42] rebuild | index.md
- NPCs: 33
- Quests: 22
- Locations: 7
- Sessions: 5
- Characters: 5
- Items: 0
- Rules: 0
- Total: 72

## [2026-04-24 10:42] lint | 371 errors, 291 warnings
- check missing-required
- check invalid-type
- check invalid-enum
- check broken-ref
- check missing-index
- check orphan
- check taxonomy
- check temporal
- check unique-slug

## [2026-04-24 10:58] lint | 371 errors, 291 warnings
- check missing-required
- check invalid-type
- check invalid-enum
- check broken-ref
- check missing-index
- check orphan
- check taxonomy
- check temporal
- check unique-slug

## [2026-04-24 11:01] rebuild | index.md
- NPCs: 33
- Quests: 22
- Locations: 7
- Sessions: 5
- Characters: 5
- Items: 0
- Rules: 3
- Total: 75

## [2026-04-24 11:01] lint | 296 errors, 291 warnings
- check missing-required
- check invalid-type
- check invalid-enum
- check broken-ref
- check missing-index
- check orphan
- check taxonomy
- check temporal
- check unique-slug

## [2026-04-24 11:01] lint | 0 errors, 1 warnings
- check missing-required
- check invalid-type
- check invalid-enum
- check broken-ref
- check missing-index
- check orphan
- check taxonomy
- check temporal
- check unique-slug

## [2026-04-24 11:02] lint | 0 errors, 1 warnings
- check missing-required
- check invalid-type
- check invalid-enum
- check broken-ref
- check missing-index
- check orphan
- check taxonomy
- check temporal
- check unique-slug
