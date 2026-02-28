---
name: rust-skills-minecraft
description: >
  Rust skill set for Minecraft-adjacent systems: voxel engines, chunk management,
  procedural world generation (Perlin / simplex / density functions), block state
  machines, entity simulation, redstone-like logic, player movement physics, NBT/SNBT
  parsing, biome interpolation, region file I/O, and async server tick loops.
  Invoke with /minecraft-rust or by mentioning any of the trigger keywords below.
  Built on the actionbook meta-cognition framework (Layer 3 → Layer 2 → Layer 1)
  and the leonardomso 179-rule taxonomy.
license: MIT
metadata:
  version: "1.0.0"
  based_on:
    - https://github.com/actionbook/rust-skills   # meta-cognition + domain-* structure
    - https://github.com/leonardomso/rust-skills   # 179-rule taxonomy
    - https://doc.rust-lang.org/stable/            # official Rust docs
    - https://rust-lang.org/learn/                 # Rust learning path
triggers:
  - voxel
  - chunk
  - block state
  - world gen
  - worldgen
  - perlin noise
  - simplex noise
  - density function
  - entity tick
  - player movement
  - redstone
  - NBT
  - SNBT
  - minecraft protocol
  - azalea
  - valence
  - pumpkin
  - feather
  - schematic
  - biome
  - tick system
  - level.dat
  - region file
  - anvil format
  - chunk loading
  - spawn chunk
  - pathfinding
  - AABB collision
  - game loop
  - minecraft server
  - voxel rendering
  - greedy meshing
---

# Rust Skills — Minecraft Domain

> Comprehensive Rust guide for Minecraft-like systems.
> Covers **Layer 0 (vibe)** + **7 sub-domains** × **3 cognitive layers** × **179-rule alignment**.

## How to Use

This skill set follows the **actionbook meta-cognition framework**:

```
Layer 0: Plain-English / Vibe   ← "I want to…" / beginner — translates to technical skills
    ↕
Layer 3: Domain Constraints (WHY)   ← Start here for architecture questions
    ↕
Layer 2: Design Choices (WHAT)      ← Patterns, data structures, crates
    ↕
Layer 1: Language Mechanics (HOW)   ← Rust ownership, lifetimes, compiler errors
```

**Routing table:**

| Your problem | Entry layer | Skills to load |
|---|---|---|
| "I want to make…" / beginner / non-technical | Layer 0 | `mc-00-vibe` → then target skill |
| E0382 in chunk system | Layer 1 → trace UP | `domain-minecraft` + `mc-01-ownership` |
| How to design chunk loading? | Layer 2 | `mc-02-chunk-storage` + `domain-minecraft` |
| Best noise algo for terrain gen? | Layer 2/3 | `mc-03-worldgen` + `domain-minecraft` |
| Entity AI slow at 10k entities | Layer 1→2 | `mc-04-entity-ecs` + `mc-07-performance` |
| Parse NBT from .mca files | Layer 1 | `mc-05-nbt-io` + `mc-01-ownership` |
| Async packet architecture | Layer 2/3 | `mc-06-networking` + `domain-minecraft` |
| Player physics jitter | Layer 2 | `mc-04-entity-ecs` + `mc-07-performance` |

---

## Skill Index

### 🦀 Entry / Vibe (Layer 0)
| Skill | Core Question | Triggers |
|---|---|---|
| [`mc-00-vibe`](skills/mc-00-vibe.md) | What do you want to build? (plain English → technical skill) | I want to make, beginner, laggy, where do I start |

### 🌍 Domain Layer (Layer 3 — WHY)
| Skill | Core Question | Triggers |
|---|---|---|
| [`domain-minecraft`](skills/domain-minecraft.md) | What are Minecraft's hard constraints? | voxel, chunk, world gen, tick, protocol |

### ⚙️ Mechanics Layer (Layer 1 — HOW)
| Skill | Core Question | Triggers |
|---|---|---|
| [`mc-01-ownership`](skills/mc-01-ownership.md) | Who owns chunk/entity data? | E0382, E0597, Arc, clone |
| [`mc-05-nbt-io`](skills/mc-05-nbt-io.md) | How to parse NBT / region files? | NBT, SNBT, .mca, region, level.dat |

### 🏗️ Design Layer (Layer 2 — WHAT)
| Skill | Core Question | Triggers |
|---|---|---|
| [`mc-02-chunk-storage`](skills/mc-02-chunk-storage.md) | What data layout for chunk data? | chunk, section, palette, block state |
| [`mc-03-worldgen`](skills/mc-03-worldgen.md) | What noise / generation algorithm? | worldgen, perlin, simplex, biome, terrain |
| [`mc-04-entity-ecs`](skills/mc-04-entity-ecs.md) | ECS vs OOP for entities? | entity, player, mob, physics, pathfinding |
| [`mc-06-networking`](skills/mc-06-networking.md) | Async server tick architecture? | server, protocol, packet, tokio, TPS |
| [`mc-07-performance`](skills/mc-07-performance.md) | Where is the 50ms budget going? | performance, lag, TPS, optimization |

---

## Cargo.toml Reference

```toml
[dependencies]
# === ECS / Game Engine ===
bevy = { version = "0.14", default-features = false, features = [
    "bevy_core_pipeline", "bevy_render", "bevy_asset"
]}

# === Minecraft Protocol / Server ===
valence = { git = "https://github.com/valence-rs/valence", optional = true }
azalea = { git = "https://github.com/azalea-rs/azalea", optional = true }

# === World Generation ===
simdnoise = "3"         # SIMD Perlin/simplex — 4-8× faster than naive
noise = "0.9"           # Fallback, pure-Rust, no SIMD

# === NBT / Data ===
fastnbt = "2"           # Fastest NBT parser, zero-copy friendly
valence_nbt = "0.8"     # Tied to valence ecosystem
flate2 = "1"            # zlib/gzip for region file decompression
memmap2 = "0.9"         # Memory-mapped region files — zero-copy reads

# === Networking ===
tokio = { version = "1", features = ["full"] }
bytes = "1"             # Zero-copy packet buffers

# === Collections / Data Structures ===
ahash = "0.8"           # Fast non-crypto HashMap — 2× faster for BlockPos keys
smallvec = "1"          # Stack-allocated small collections (palette, neighbours)
bitvec = "1"            # Bit-packed arrays for paletted containers
glam = "0.28"           # SIMD Vec3/Mat4 — used internally by Bevy

# === Spatial / Math ===
rustc-hash = "2"        # Ultra-fast hash for integer spatial keys

[profile.release]
opt-level = 3
lto = "fat"
codegen-units = 1
panic = "abort"

[profile.dev.package."*"]
opt-level = 3           # Keep dependencies fast in dev
```

---

## Rule Alignment (leonardomso taxonomy)

The following leonardomso rules apply with **Minecraft-specific nuance**:

| Rule | Minecraft context |
|---|---|
| `own-arc-shared` | `Arc<RwLock<Chunk>>` — chunks shared by world, renderer, entities |
| `own-rwlock-readers` | Chunk reads dominate writes; use `RwLock`, not `Mutex` |
| `mem-with-capacity` | Pre-size `Vec<BlockState>` to `4096` per section |
| `mem-smallvec` | Palette entries per section rarely exceed 16 |
| `mem-boxed-slice` | `Box<[u8; 2048]>` for light arrays — fixed-size, no realloc |
| `async-no-lock-await` | NEVER hold a chunk `RwLock` across `.await` — server deadlock |
| `async-spawn-blocking` | Chunk generation is CPU-bound → `spawn_blocking` |
| `async-bounded-channel` | Backpressure on packet ingestion — bounded `mpsc` |
| `opt-simd-portable` | `simdnoise` for terrain generation — mandatory |
| `opt-cache-friendly` | SoA layout for entity components via ECS |
| `perf-iter-over-index` | Iterate chunk sections with `par_iter()` (rayon) |
| `anti-unwrap-abuse` | Chunk lookups return `Option<Chunk>` — never `.unwrap()` |
| `type-newtype-ids` | `ChunkPos(i32, i32)`, `EntityId(u32)`, `BlockState(u16)` |

---

## Sources

- [Rust Programming Language Book](https://doc.rust-lang.org/book/)
- [Rust Standard Library](https://doc.rust-lang.org/stable/std/)
- [Rust Performance Book](https://nnethercote.github.io/perf-book/)
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)
- [wiki.vg — Minecraft Protocol](https://wiki.vg/Protocol)
- [Minecraft Wiki — Chunk Format](https://minecraft.wiki/w/Chunk_format)
- [valence-rs/valence](https://github.com/valence-rs/valence)
- [azalea-rs/azalea](https://github.com/azalea-rs/azalea)
