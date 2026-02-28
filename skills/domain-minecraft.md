---
name: domain-minecraft
description: >
  Layer 3 — Domain Constraints (WHY). The anchor skill for any Minecraft-adjacent
  Rust work. Defines hard constraints that must be respected before any design or
  language decision. Always load this first, then trace DOWN to Layer 2 / Layer 1.
layer: 3
triggers:
  - voxel, chunk, block state, world gen, entity tick, redstone, NBT, minecraft server
  - azalea, valence, pumpkin, minecraft protocol, TPS, tick budget
---

# Domain: Minecraft / Voxel Systems in Rust

> **Meta-Cognition Entry (Layer 3 → trace DOWN)**
>
> Don't answer "what Rust type to use" until you've answered:
> 1. **What is the tick budget?** (50 ms = 20 TPS — non-negotiable)
> 2. **Is this server-side or client-side?** (different latency profiles)
> 3. **How big is the spatial scale?** (30M × 30M × 384 blocks — far beyond a `HashMap`)

---

## Hard Domain Constraints

These are facts about Minecraft's simulation model. Every design and language decision
must respect them — they cannot be optimized away.

| Constraint | Value | Rust implication |
|---|---|---|
| Tick budget | 50 ms @ 20 TPS | No heap alloc in hot tick path; no blocking I/O on tick thread |
| Chunk column | 16×16×384 blocks | Packed `[u16; 4096]` per section beats `HashMap<BlockPos, Block>` |
| Build height range | Y = −64 to +320 | Always use **`i32` for Y coordinates** — never `u32` |
| Spatial scale | ±30,000,000 blocks on X/Z | Use **`f64` for world coords** — `f32` loses precision beyond ±2048 blocks |
| Block state count | ~25,000 in 1.21 | `BlockState(u16)` fits all states; global palette fits in a `u16` array |
| Max entities per chunk | 1–10k+ (servers vary) | ECS mandatory — OOP inheritance collapses under composition pressure |
| Redstone tick order | Deterministic BFS wave | Update queue, not hashset — order matters for technical Minecraft |
| Region file format | Anvil (4KB sector aligned) | Memory-map with `memmap2`; never `read_to_end` large .mca files |
| Protocol compression | zlib at threshold | `flate2` + zero-copy `Bytes` for packet buffers |
| UUID format | 128-bit, hyphenated string | Use `uuid` crate — manual parsing introduces subtle bugs |

---

## Ecosystem Map

```
── minecraft server / protocol ──────────────────────────────────────────────
   valence          Full Minecraft server framework, ECS-native, extensible
   pumpkin          Lightweight async server (actively maintained, 2024+)
   azalea           Full client in Rust: auth, protocol, world simulation
   ferrumc          Clean-room ECS server, perf-first, plugin FFI roadmap (2024+)
   ⚠️  feather      ABANDONED — do not use as reference for new projects

── bedrock edition ──────────────────────────────────────────────────────────
   bedrock-rs       Universal toolkit for Minecraft Bedrock development in Rust
                    (protocol, NBT, world format — separate from Java edition)

── voxel / rendering ────────────────────────────────────────────────────────
   bevy             ECS game engine — most production clones target this
   wgpu             Cross-platform GPU API for chunk mesh rendering
   block-mesh-rs    Greedy / simple meshing algorithms for voxel faces

── world generation ─────────────────────────────────────────────────────────
   simdnoise        SIMD Perlin + simplex — 4–8× faster than scalar
   noise            Pure-Rust reference, portable, no SIMD
   splines          Terrain spline interpolation (Minecraft 1.18+ style)

── data / serialization ─────────────────────────────────────────────────────
   fastnbt          Fastest NBT parser; zero-copy where possible
   valence_nbt      NBT tied to valence ecosystem
   flate2           zlib / gzip for region file decompression

── networking ───────────────────────────────────────────────────────────────
   tokio            Async runtime for server networking (mandatory)
   bytes            Zero-copy Bytes buffers for packet I/O

── lighting ─────────────────────────────────────────────────────────────────
   (no crate — implement from spec; see mc-08-lighting)
   Reference: Starlight (PaperMC) TECHNICAL_DETAILS.md
   Reference: dktapps/lighting-algorithm-spec

── plugin / extension ───────────────────────────────────────────────────────
   libloading       Dynamic .so/.dll loading (unsafe, same-platform)
   wasmtime         WASM sandbox for safe cross-platform plugins
   wasmtime-wasi    WASI context (controls plugin filesystem/network access)
```

---

## Cognitive Layer Routing

When you receive a Minecraft Rust problem, trace through:

```
                     ┌─────────────────────────────────────┐
                     │  Layer 3: Domain Constraints (WHY)  │  ← You are here
                     │  domain-minecraft                   │
                     └──────────────┬──────────────────────┘
                                    │ trace DOWN
                     ┌──────────────▼──────────────────────┐
                     │  Layer 2: Design Choices (WHAT)     │
                     │  mc-02 chunk · mc-03 worldgen        │
                     │  mc-04 ECS   · mc-06 networking      │
                     │  mc-07 perf  · mc-08 lighting        │
                     │  mc-10 plugins                       │
                     └──────────────┬──────────────────────┘
                                    │ trace DOWN
                     ┌──────────────▼──────────────────────┐
                     │  Layer 1: Language Mechanics (HOW)  │
                     │  mc-01 ownership · mc-05 NBT I/O    │
                     └─────────────────────────────────────┘
```

**Routing signals:**

| Signal | Entry Layer | Primary Skill |
|---|---|---|
| E0382, E0597, borrow error in chunk code | Layer 1 → UP | `mc-01-ownership` |
| "How to design chunk loading" | Layer 2 | `mc-02-chunk-storage` |
| "Best noise for terrain" | Layer 2/3 | `mc-03-worldgen` |
| "Entity AI slow" | Layer 1→2 | `mc-04-entity-ecs` + `mc-07-performance` |
| "Parse NBT from .mca" | Layer 1 | `mc-05-nbt-io` |
| "Async packet server" | Layer 2/3 | `mc-06-networking` |
| "Server lags, TPS drops" | Layer 1→2 | `mc-07-performance` |
| "Dark chunks, light leaks, skylight wrong" | Layer 2 | `mc-08-lighting` |
| "Want plugin system, extensible server" | Layer 2 | `mc-10-plugins` |
| "I don't know Rust, where do I start" | Layer 0 | `mc-00-vibe` |

---

## Domain-Critical Type Vocabulary

These newtypes should exist in any non-trivial Minecraft Rust codebase:

```rust
// src/types.rs

/// Absolute chunk position in the world grid
#[derive(Debug, Clone, Copy, Hash, PartialEq, Eq)]
pub struct ChunkPos { pub x: i32, pub z: i32 }

/// Block position — Y can be negative (−64 in 1.18+)
#[derive(Debug, Clone, Copy, Hash, PartialEq, Eq)]
pub struct BlockPos { pub x: i32, pub y: i32, pub z: i32 }

/// A block state ID in the global palette (0..=max_state_id)
/// Fits in u16 — Minecraft 1.21 has ~25,000 states
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub struct BlockState(pub u16);

/// A biome registry ID
#[derive(Debug, Clone, Copy, PartialEq, Eq, Default)]
pub struct BiomeId(pub u32);

/// Entity runtime ID — scoped to server lifetime, not persisted
#[derive(Debug, Clone, Copy, Hash, PartialEq, Eq)]
pub struct EntityId(pub u32);

/// Server tick counter — wraps at u64::MAX (~29 million years at 20 TPS)
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub struct Tick(pub u64);

impl ChunkPos {
    /// Pack into u64 for fast AHashMap keys
    #[inline] pub fn as_u64(self) -> u64 {
        ((self.x as u64) << 32) | (self.z as u32) as u64
    }
    /// Get the chunk position from a block position
    pub fn from_block(pos: BlockPos) -> Self {
        Self { x: pos.x >> 4, z: pos.z >> 4 }
    }
}

impl BlockPos {
    /// Local block index within a chunk section (0..4096)
    #[inline] pub fn section_index(self) -> usize {
        let lx = (self.x & 15) as usize;
        let ly = (self.y & 15) as usize;
        let lz = (self.z & 15) as usize;
        ly << 8 | lz << 4 | lx   // Y-major for skylight propagation
    }
}
```

> **Why newtypes?** `i32` and `i32` are both valid X coordinates and entity IDs.
> The compiler cannot help you if they're the same type. `ChunkPos(i32, i32)` is
> a zero-cost abstraction — no runtime overhead, massive compile-time safety.

---

## The 50ms Tick Budget — Detailed

```
Typical server tick breakdown (100 players, 20×20 chunk radius each):

  Entity tick          ~10 ms   (10,000 entities × movement + collision)
  Chunk random tick    ~4 ms    (crops, fire spread, ice melting)
  Block entity tick    ~5 ms    (furnaces, hoppers, chests with listeners)
  Redstone propagation ~3 ms    (complex circuits)
  Player packet I/O    ~0 ms    (async — separate thread pool)
  Chunk load/save      ~0 ms    (async — separate thread pool)
  Lighting updates     ~5 ms    (sky + block light propagation)
  World border checks  ~1 ms
  ─────────────────────────
  Headroom            ~22 ms    (used by GC-free Rust vs Java)

Rule: If any single system exceeds 10ms, investigate immediately.
```

---

## Common Pitfalls (Domain Level)

| Pitfall | Consequence | Correct approach |
|---|---|---|
| `f32` for world coordinates | Precision loss > ±2048 blocks ("far lands" equivalent) | `f64` world coords; `f32` only for rendering deltas |
| `u32` for Y coordinate | Panic / underflow at Y < 0 (1.18+ baseline is Y = −64) | `i32` always for Y |
| `HashMap` per block | Cache miss flood on chunk iteration | Packed `[u16; 4096]` per section |
| Blocking I/O on tick thread | Server freezes during chunk load/save | `tokio::spawn_blocking` for all file I/O |
| Locking chunk data across `.await` | Async deadlock under load | Release `RwLock` before any `.await` point |
| `String` for block IDs at runtime | Alloc + hash every block lookup | Intern to `u16` registry ID at startup |
| Naive redstone scan per tick | O(chunk_volume) = 98,304 iterations for nothing | BFS dirty-flag queue |
