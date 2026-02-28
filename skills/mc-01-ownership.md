---
name: mc-01-ownership
description: >
  Layer 1 — Language Mechanics (HOW). Ownership and borrowing patterns specific to
  Minecraft-like Rust systems. Covers Arc<RwLock<Chunk>>, entity ID indirection,
  cross-system data access, and ECS component ownership. Use when you hit borrow
  errors (E0382, E0597, E0499) in chunk, entity, or world-level code.
layer: 1
triggers:
  - E0382, E0597, E0499, borrow, move, clone chunk, Arc chunk, RwLock world
---

# mc-01 — Ownership in Minecraft Systems

> **Core Question**: *Who owns chunk data — the world, the player, the renderer?*
>
> Minecraft has 3 fundamental ownership tensions:
> 1. **Chunks** are read by many (players, renderer, entities) but written by few
> 2. **Entities** live in chunks but cross chunk boundaries (movement)
> 3. **World** is singleton, but systems run in parallel

---

## Pattern Decision Tree

```
Is data written by multiple threads at once?
├── YES → RwLock<T>  (prefer over Mutex when reads dominate writes)
│         Chunk data: many readers (rendering, entity queries), rare writers
└── NO  → Is it shared across ownership boundaries (world ↔ system)?
          ├── YES, multiple owners → Arc<T>
          │   Example: Arc<RwLock<Chunk>> — owned by World AND by ChunkRenderer
          └── NO → plain T or &T borrow
                   Example: &ChunkSection passed into a meshing function
```

---

## Rule: `mc-own-chunk` — Use `Arc<RwLock<Chunk>>` for loaded chunks

**Why**: Chunks are accessed from the tick system (write), the renderer (read), entity
queries (read), and async chunk loaders (write). Multiple independent owners, mixed
read/write access profile.

```rust
// ❌ Wrong: single ownership prevents sharing
struct World {
    chunks: HashMap<ChunkPos, Chunk>,  // borrowed out → can't loan to renderer
}

// ✅ Correct: reference-counted shared ownership with read/write semantics
use std::sync::{Arc, RwLock};
use ahash::AHashMap;

pub struct World {
    // Arc: multiple owners (tick system + renderer + async loader all hold refs)
    // RwLock: many readers, rare writers → better throughput than Mutex
    chunks: AHashMap<u64, Arc<RwLock<Chunk>>>,
}

impl World {
    pub fn get_chunk(&self, pos: ChunkPos) -> Option<Arc<RwLock<Chunk>>> {
        self.chunks.get(&pos.as_u64()).cloned()  // cheap Arc clone, not Chunk clone
    }
}

// Usage — always release lock before any async boundary
fn process_chunk(world: &World, pos: ChunkPos) {
    let chunk_arc = world.get_chunk(pos)?;
    let block = {
        let chunk = chunk_arc.read().unwrap();  // acquire read lock
        chunk.get_block(0, 64, 0)
    };  // <── lock released HERE, before any .await
    // ... use block
}
```

> **leonardomso rule**: `own-rwlock-readers` — use `RwLock<T>` when reads dominate writes.
> `own-arc-shared` — use `Arc<T>` for thread-safe shared ownership.

---

## Rule: `mc-own-entity-id` — Entities are IDs, not pointers

Holding a direct reference to an entity across tick boundaries causes lifetime hell.
The correct pattern: entities are identified by `EntityId(u32)`, looked up per tick.

```rust
// ❌ Wrong: lifetime ties entity to world borrow
struct Arrow<'w> {
    shooter: &'w Entity,  // E0597: shooter won't live long enough across ticks
}

// ✅ Correct: entity indirection via ID
pub struct Arrow {
    shooter_id: EntityId,  // just a u32, no lifetime, freely Copy-able
    position: DVec3,       // f64 world coords — see domain-minecraft.md
    velocity: DVec3,
}

// Resolve at tick time — cheap HashMap lookup
fn tick_arrow(arrow: &mut Arrow, world: &World) {
    if let Some(shooter) = world.entity(arrow.shooter_id) {
        // ... use shooter for this tick only
    }
    // shooter reference dropped at end of scope — no lifetime leakage
}
```

> **leonardomso rule**: `type-newtype-ids` — wrap IDs in newtypes.

---

## Rule: `mc-own-no-lock-await` — NEVER hold chunk lock across `.await`

This is the most common async deadlock pattern in Minecraft server code.

```rust
// ❌ DEADLOCK: lock held across .await
async fn save_chunk(world: &World, pos: ChunkPos) {
    let chunk = world.get_chunk(pos).unwrap();
    let guard = chunk.read().unwrap();  // lock acquired

    write_to_disk(&guard).await;        // <── .await while lock held!
    //                                       Other tasks that need this chunk FREEZE
}

// ✅ Correct: clone data needed for I/O, release lock first
async fn save_chunk(world: &World, pos: ChunkPos) {
    let chunk_data = {
        let chunk = world.get_chunk(pos).unwrap();
        let guard = chunk.read().unwrap();
        guard.to_nbt()          // clone/serialize while holding lock
    };                          // <── lock released HERE

    write_nbt_to_disk(chunk_data).await;  // safe — no lock held
}
```

> **leonardomso rule**: `async-no-lock-await` — never hold Mutex/RwLock across `.await`.

---

## Rule: `mc-own-cow-block-names` — Use `Cow<str>` for block namespaced IDs

Block names like `"minecraft:stone"` are usually known at compile time but occasionally
constructed dynamically (modded content, data packs).

```rust
use std::borrow::Cow;

// ❌ Wrong: always allocates, even for literals
fn register_block(name: String) { /* ... */ }
register_block("minecraft:stone".to_string()); // needless alloc

// ✅ Correct: zero-cost for literals, works for dynamic names
fn register_block(name: Cow<'static, str>) { /* ... */ }
register_block(Cow::Borrowed("minecraft:stone")); // zero alloc
register_block(Cow::Owned(format!("{}:{}", ns, id))); // alloc only when needed
```

> **leonardomso rule**: `own-cow-conditional` — use `Cow<'a, T>` for conditional ownership.

---

## Common Ownership Errors in Minecraft Code

| Error | Typical cause | Fix |
|---|---|---|
| E0382 (use after move) | Passing `Chunk` into a closure that captures it | Use `Arc<Chunk>` + `.clone()` the Arc |
| E0597 (borrowed value does not live long enough) | Storing `&Entity` in a struct | Replace with `EntityId(u32)` |
| E0499 (cannot borrow as mutable, also borrowed as immutable) | Accessing `world.entities` while iterating | Use index-based iteration or ECS |
| E0502 (cannot borrow as mutable because also borrowed) | Mutating one chunk while reading another | `DashMap` or fine-grained `RwLock` per chunk |
