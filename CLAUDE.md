# Rust Skills — Minecraft Domain (CLAUDE.md)

## Mandatory Routing Rule

For ANY Minecraft / voxel Rust question, ALWAYS follow this chain:

```
[1] Load: domain-minecraft  ← establishes hard domain constraints (Layer 3)
[2] Identify problem type → route to Layer 2 or Layer 1 skill
[3] Answer with domain-correct solution, not surface-level fix
```

**Do NOT** jump straight to "use Arc" or "add .clone()" — first ask WHY the data is shared.

---

## Quick Routing Table

| User signal | Skill chain |
|---|---|
| E0382 / E0597 in chunk or entity code | `domain-minecraft` → `mc-01-ownership` |
| "How to store chunk data" | `domain-minecraft` → `mc-02-chunk-storage` |
| "How to generate terrain / noise" | `domain-minecraft` → `mc-03-worldgen` |
| "Entity movement / physics / pathfinding" | `domain-minecraft` → `mc-04-entity-ecs` |
| "Parse NBT / region files / .mca" | `domain-minecraft` → `mc-05-nbt-io` |
| "Async server / packet handling / TPS" | `domain-minecraft` → `mc-06-networking` |
| "Server is lagging / TPS drops / slow" | `domain-minecraft` → `mc-07-performance` |
| "Redstone logic" | `domain-minecraft` → `mc-07-performance` (BFS section) |
| "Player jump / sprint / sneak physics" | `domain-minecraft` → `mc-04-entity-ecs` |
| "World seed / biome blending" | `domain-minecraft` → `mc-03-worldgen` |

---

## Meta-Cognition Protocol

When answering a Minecraft Rust question, trace through:

```
Layer 3 (WHY): What Minecraft domain constraint applies here?
    → 50ms tick? Chunk layout? Protocol requirement? Spatial scale?
Layer 2 (WHAT): What design pattern / data structure satisfies that constraint?
    → Paletted container? ECS? BFS queue? Async channel?
Layer 1 (HOW): What Rust mechanics implement that pattern?
    → Arc<RwLock>? par_iter? simdnoise batch? fastnbt::from_bytes?
```

**Example:**
```
User: "My chunk system uses HashMap<BlockPos, BlockState> and it's slow"

Layer 3: Chunk iteration accesses blocks in Y/Z/X order → cache locality matters
Layer 2: Use packed section array [u16; 4096] with Y-major index, not HashMap
Layer 1: PalettedContainer<BlockState, 4096> — indirect palette for memory efficiency

Answer: Replace HashMap with paletted section arrays (see mc-02-chunk-storage)
NOT: "Use a faster HashMap like AHashMap"  ← surface fix, wrong architecture
```

---

## Cargo.toml Default Settings

When setting up a Minecraft Rust project:

```toml
[package]
edition = "2024"
rust-version = "1.85"

[lints.rust]
unsafe_code = "warn"

[lints.clippy]
all = "warn"
pedantic = "warn"

[profile.release]
opt-level = 3
lto = "fat"
codegen-units = 1
panic = "abort"

[profile.dev.package."*"]
opt-level = 3  # simdnoise needs this in dev
```

---

## Key Domain Facts (memorize these)

1. **Y coordinates can be negative** (−64 in 1.18+) → always `i32`, never `u32`
2. **50ms tick = 20 TPS** — never block tick thread on I/O
3. **f64 for world coords** — f32 loses precision > ±2048 blocks
4. **Chunk = 24 sections** (1.18+, −64 to +320 build height)
5. **Section = 16×16×16 = 4096 blocks** — index as `y*256 + z*16 + x`
6. **BlockState fits u16** — max ~25,000 states in 1.21
7. **Never hold RwLock across .await** — async deadlock
8. **simdnoise batch per section** — never per-block noise calls
9. **VarInt prefix on every Minecraft packet** — MSB = more bytes, 7 bits per byte
10. **Redstone = BFS dirty-flag queue** — not per-tick full scan
