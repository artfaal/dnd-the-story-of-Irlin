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
