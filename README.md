# Rust Skills — Minecraft Domain

> AI coding assistant skill set for Minecraft-adjacent Rust systems.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

## What's in here

**8 skill files** across 3 cognitive layers covering every major Minecraft Rust subsystem:

| Layer | Skills | What it covers |
|---|---|---|
| **Layer 3 — Domain** | `domain-minecraft` | Hard constraints: 50ms tick, coord types, ecosystem map |
| **Layer 1 — Mechanics** | `mc-01-ownership`, `mc-05-nbt-io` | Borrow errors, chunk ownership, NBT/region file I/O |
| **Layer 2 — Design** | `mc-02` through `mc-07` | Chunk layout, worldgen, ECS/physics, networking, perf |

## Install

**Claude Code (global):**
```bash
git clone https://github.com/your-org/rust-skills-minecraft ~/.claude/skills/rust-skills-minecraft
```

**Claude Code (project only):**
```bash
git clone https://github.com/your-org/rust-skills-minecraft .claude/skills/rust-skills-minecraft
```

**Any agent (AGENTS.md standard):**
```bash
cp rust-skills-minecraft/AGENTS.md AGENTS.md
```

**Cursor:**
```bash
cp rust-skills-minecraft/AGENTS.md .cursorrules
```

## What it covers

| Subsystem | Skill | Key patterns |
|---|---|---|
| Chunk storage | `mc-02-chunk-storage` | Paletted containers, Y-major index, nibble arrays |
| World generation | `mc-03-worldgen` | 2D+3D noise chains, SIMD batching, two-phase gen |
| Entity simulation | `mc-04-entity-ecs` | Bevy ECS, AABB physics, A* pathfinding |
| NBT / region I/O | `mc-05-nbt-io` | memmap2, fastnbt, typed serde structs |
| Async networking | `mc-06-networking` | VarInt, protocol state machine, tick separation |
| Performance | `mc-07-performance` | Greedy meshing, BFS redstone, AHashMap, pooling |
| Ownership | `mc-01-ownership` | Arc<RwLock<Chunk>>, entity ID indirection |

## Based on

- [actionbook/rust-skills](https://github.com/actionbook/rust-skills) — meta-cognition framework (Layer 3 → 2 → 1)
- [leonardomso/rust-skills](https://github.com/leonardomso/rust-skills) — 179-rule taxonomy
- [doc.rust-lang.org/stable](https://doc.rust-lang.org/stable/) — official Rust documentation
- [rust-lang.org/learn](https://rust-lang.org/learn/) — Rust learning path

## License

MIT
