# User Guide — Rust Skills Minecraft Domain

Last updated: 2026-02-28

## What this is

This repo is a **skill set** for AI assistants and developers working on Minecraft-adjacent Rust systems (voxel engines, chunk storage, worldgen, entities, NBT I/O, networking). It provides structured rules and patterns so answers stay domain-correct.

## How to use it

1. **Install** the skill set in your environment (see [README.md](README.md) — Claude Code, Cursor, Copilot, or copy `AGENTS.md`).
2. **If you're new or prefer plain English**, say what you want in words (e.g. *"I want to make a world generator"*, *"my server is laggy"*). The assistant should load **`mc-00-vibe`** first — it maps your goal to the right technical skill. See [skills/mc-00-vibe.md](skills/mc-00-vibe.md) for the full "I want to…" guide.
3. **When asking with technical terms**, mention voxel, chunk, worldgen, entity, NBT, protocol, TPS, etc. The assistant loads `domain-minecraft` first, then routes to the right `mc-*` skill.
4. **Routing**: Use [CLAUDE.md](CLAUDE.md) or [SKILL.md](SKILL.md) to see which skill covers your topic.

## Quick reference

| You want to…              | Load this skill              |
|---------------------------|------------------------------|
| **"I want to make…" / beginner** | **`mc-00-vibe`** → then target skill below |
| Fix E0382/E0597 in chunks | `domain-minecraft` + `mc-01-ownership` |
| Design chunk storage      | `mc-02-chunk-storage`        |
| Terrain / noise           | `mc-03-worldgen`             |
| Entity movement / ECS     | `mc-04-entity-ecs`           |
| Parse NBT / .mca files   | `mc-05-nbt-io`               |
| Async server / packets    | `mc-06-networking`           |
| Fix lag / TPS             | `mc-07-performance`          |
| Lighting / skylight / light leaks | `mc-08-lighting`     |
| Plugin system / extensible server | `mc-10-plugins`      |

For full install and layout, see [INSTALL.md](INSTALL.md) and [README.md](README.md).
