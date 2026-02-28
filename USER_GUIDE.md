# User Guide — Rust Skills Minecraft Domain

Last updated: 2026-02-28

## What this is

This repo is a **skill set** for AI assistants and developers working on Minecraft-adjacent Rust systems (voxel engines, chunk storage, worldgen, entities, NBT I/O, networking). It provides structured rules and patterns so answers stay domain-correct.

## How to use it

1. **Install** the skill set in your environment (see [README.md](README.md) — Claude Code, Cursor, Copilot, or copy `AGENTS.md`).
2. **When asking about Minecraft Rust code**, mention one of: voxel, chunk, worldgen, entity, NBT, protocol, TPS, etc. The assistant should load `domain-minecraft` first, then route to the right `mc-*` skill.
3. **Routing**: Use [CLAUDE.md](CLAUDE.md) or [SKILL.md](SKILL.md) to see which skill covers your topic (e.g. borrow errors → `mc-01-ownership`, terrain generation → `mc-03-worldgen`).

## Quick reference

| You want to…              | Load this skill              |
|---------------------------|------------------------------|
| Fix E0382/E0597 in chunks | `domain-minecraft` + `mc-01-ownership` |
| Design chunk storage      | `mc-02-chunk-storage`        |
| Terrain / noise           | `mc-03-worldgen`             |
| Entity movement / ECS     | `mc-04-entity-ecs`           |
| Parse NBT / .mca files   | `mc-05-nbt-io`               |
| Async server / packets    | `mc-06-networking`           |
| Fix lag / TPS             | `mc-07-performance`          |

For full install and layout, see [INSTALL.md](INSTALL.md) and [README.md](README.md).
