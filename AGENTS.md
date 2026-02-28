# Rust Skills — Minecraft Domain

> AI coding assistant rules for Minecraft-adjacent Rust systems.
> Derived from actionbook/rust-skills (meta-cognition framework) and
> leonardomso/rust-skills (179-rule taxonomy), specialized for voxel game development.

## Default Project Setup

```toml
[package]
edition = "2024"
rust-version = "1.85"

[lints.rust]
unsafe_code = "warn"

[lints.clippy]
all = "warn"
pedantic = "warn"
```

## Core Domain Rules (always apply)

### Coordinates & Types
- **ALWAYS use `i32` for Y coordinates** — build height is −64 to +320 in 1.18+
- **ALWAYS use `f64` for world coordinates** — `f32` loses precision beyond ±2048 blocks
- **ALWAYS use `BlockState(u16)` newtype** — not bare `u16`, not `String`
- **ALWAYS use `ChunkPos { x: i32, z: i32 }` newtype** — not `(i32, i32)` tuple

### Chunk Storage
- **NEVER use `HashMap<BlockPos, BlockState>`** for chunk block data — use packed `[u16; 4096]`
- **ALWAYS use paletted containers** — single-value optimization for uniform sections (mostly air)
- **ALWAYS index with Y-major layout**: `index = y*256 + z*16 + x` for skylight cache efficiency
- **ALWAYS use `Box<[u8; 2048]>` for light arrays** — fixed size, nibble-packed, no realloc

### World Generation
- **NEVER call noise per-block** — ALWAYS batch by section (4096 values per SIMD call)
- **ALWAYS use `simdnoise`** for performance-critical generation paths
- **ALWAYS use two-phase generation**: Phase 1 = base terrain (parallel), Phase 2 = decoration (sequential)
- **ALWAYS seed with `u64`** and use deterministic noise (same seed → same world)

### Entity Simulation
- **NEVER use OOP inheritance for entities** — ALWAYS use ECS (Bevy recommended)
- **NEVER store `&Entity` references** — use `EntityId(u32)` and resolve per tick
- **ALWAYS apply gravity as `vel.y -= 0.08; vel.y *= 0.98;`** (Minecraft constants)
- **ALWAYS use AABB sweep test** for collision — never teleport through blocks
- **ALWAYS resolve Y-axis collision before X and Z** (Minecraft sweep order)

### Async & Networking
- **NEVER block the tick thread** — all I/O must be `tokio::spawn` or `spawn_blocking`
- **NEVER hold `RwLock` or `Mutex` across `.await`** — async deadlock
- **ALWAYS use bounded `mpsc` channels** for packet ingestion — backpressure is mandatory
- **ALWAYS use `tokio::time::interval(Duration::from_millis(50))`** for the tick loop
- **ALWAYS implement VarInt** for Minecraft packet framing (7 bits per byte, MSB = continue)

### Performance
- **ALWAYS use `AHashMap`** over `std::HashMap` for spatial lookups — ~2× faster for int keys
- **ALWAYS pre-size collections**: `AHashMap::with_capacity(expected)` for chunk maps
- **ALWAYS use BFS dirty-flag queue** for redstone — never O(n) per-tick block scan
- **ALWAYS use greedy meshing** (`block-mesh-rs`) — reduces quad count by ~95%
- **ALWAYS use `par_iter()`** (rayon) for chunk generation — embarrassingly parallel

### Error Handling
- **NEVER `.unwrap()` on chunk lookups** — missing chunks are expected, not bugs
- **ALWAYS return `Result<Chunk, ChunkReadError>`** from I/O functions
- **ALWAYS use `thiserror`** for domain error types

### NBT & File I/O
- **ALWAYS memory-map `.mca` files** with `memmap2` — never `read_to_end` region files
- **ALWAYS use `fastnbt`** for NBT parsing — fastest available
- **ALWAYS use typed serde structs** for NBT deserialization — not `Value::get()`

## Skill Files

See `skills/` directory for detailed guidance:
- `domain-minecraft.md` — Domain constraints and routing
- `mc-01-ownership.md` — Arc/RwLock/entity ID patterns
- `mc-02-chunk-storage.md` — Paletted containers, section layout
- `mc-03-worldgen.md` — Noise chains, density functions
- `mc-04-entity-ecs.md` — Physics, player input, pathfinding
- `mc-05-nbt-io.md` — NBT parsing, region file format
- `mc-06-networking.md` — Async server, protocol state machine
- `mc-07-performance.md` — Profiling, greedy mesh, BFS redstone

## Sources
- https://doc.rust-lang.org/stable/
- https://rust-lang.org/learn/
- https://github.com/actionbook/rust-skills
- https://github.com/leonardomso/rust-skills
- https://wiki.vg/Protocol
- https://minecraft.wiki/w/Chunk_format
