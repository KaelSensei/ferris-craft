---
name: mc-07-performance
description: >
  Layer 2 — Design Choices (WHAT). Minecraft-specific performance optimization:
  spatial indexing, object pooling, chunk meshing (greedy meshing), redstone BFS,
  parallel chunk generation, and tick budget profiling. Use when TPS drops, entity
  tick is slow, or chunk gen/mesh perf needs improvement.
layer: 2
triggers:
  - performance, lag, TPS drop, optimization, slow, profiling, benchmark
  - greedy meshing, chunk mesh, redstone lag, entity lag, tick overrun
---

# mc-07 — Performance Optimization

> **Core Question**: *Where in the 50ms tick budget is time being lost?*
>
> **Profile first. Never optimize without data.**
> Use `tracing` + flamegraph to identify the system, then apply the fix below.

---

## Profiling Setup

```rust
// Add to main (dev only):
use tracing_subscriber::layer::SubscriberExt;

fn init_tracing() {
    let subscriber = tracing_subscriber::registry()
        .with(tracing_subscriber::fmt::layer())
        .with(tracing_subscriber::EnvFilter::from_default_env());
    tracing::subscriber::set_global_default(subscriber).unwrap();
}

// Instrument every major tick system:
fn tick_entities(world: &mut World) {
    let _span = tracing::info_span!("entity_tick").entered();
    // ... your code
}

// cargo flamegraph -- --your-server-args
// Look for: entity_tick, chunk_gen, lighting, redstone > 10ms
```

---

## Rule: `mc-perf-ahash` — Use AHashMap for all spatial lookups

```rust
// ❌ Wrong: std HashMap uses SipHash — DoS-resistant but slow for int keys
use std::collections::HashMap;
let mut chunks: HashMap<u64, Chunk> = HashMap::new();

// ✅ Correct: AHashMap — ~2× faster for integer/pointer keys
use ahash::AHashMap;
let mut chunks: AHashMap<u64, Arc<RwLock<Chunk>>> = AHashMap::new();

// Even better: pre-size if you know approximate capacity
let mut chunks = AHashMap::with_capacity(1024);  // typical loaded chunk count
```

> **leonardomso rule**: `mem-with-capacity` — pre-size collections when size is known.

---

## Rule: `mc-perf-object-pool` — Pool generation scratch buffers

Chunk generation allocates the same large scratch buffers repeatedly.
Pooling eliminates the alloc/dealloc overhead in the hot gen path.

```rust
use std::sync::Mutex;

/// Pool of reusable noise scratch buffers.
/// Avoids 98,304 × sizeof(f32) allocation per chunk on every gen call.
pub struct NoiseBufferPool {
    pool: Mutex<Vec<Box<[f32; 4096]>>>,
}

impl NoiseBufferPool {
    pub fn new() -> Self { Self { pool: Mutex::new(Vec::new()) } }

    pub fn acquire(&self) -> PooledBuffer<'_> {
        let buf = self.pool.lock().unwrap().pop()
            .unwrap_or_else(|| Box::new([0.0f32; 4096]));
        PooledBuffer { buf, pool: self }
    }
}

pub struct PooledBuffer<'a> {
    pub buf: Box<[f32; 4096]>,
    pool: &'a NoiseBufferPool,
}

impl Drop for PooledBuffer<'_> {
    fn drop(&mut self) {
        let buf = std::mem::replace(&mut self.buf, Box::new([0.0f32; 4096]));
        self.pool.pool.lock().unwrap().push(buf);
    }
}

// Usage in gen pipeline:
fn generate_section(pool: &NoiseBufferPool, gen: &TerrainGenerator, pos: ChunkPos, y: i8) {
    let mut scratch = pool.acquire();
    gen.fill_density_buffer(&mut scratch.buf, pos, y);
    // process scratch.buf...
    // drop(scratch) → buffer returned to pool automatically
}
```

---

## Rule: `mc-perf-greedy-meshing` — Greedy mesh for chunk rendering

Naive voxel meshing = 6 quads per solid block = 6 × 4096 = 24,576 quads max per section.
Greedy meshing merges coplanar same-material faces → typically 95% reduction.

```rust
// Use block-mesh-rs crate — production-quality implementation
use block_mesh::{greedy_quads, GreedyQuadsBuffer, RIGHT_HANDED_Y_UP_CONFIG, MergeVoxel};

#[derive(Clone, Copy, PartialEq)]
struct VoxelData(BlockState);

impl MergeVoxel for VoxelData {
    type MergeValue = BlockState;
    fn merge_value(&self) -> BlockState { self.0 }
}

pub fn mesh_chunk_section(section: &ChunkSection) -> Vec<Quad> {
    // 18×18×18 padded array (1-block border from neighbours for face culling)
    let padded: Vec<VoxelData> = build_padded_section(section);

    let mut buffer = GreedyQuadsBuffer::new(padded.len());

    greedy_quads(
        &padded,
        &RIGHT_HANDED_Y_UP_CONFIG,   // Y-up coordinate system
        [0u32; 3],                    // min
        [17u32; 3],                   // max (16 + 1 padding)
        &RIGHT_HANDED_Y_UP_CONFIG.faces,
        &mut buffer,
    );

    // Convert quads to vertex data for GPU upload
    buffer.quads.groups.iter()
        .flat_map(|group| group.iter())
        .map(|quad| Quad::from_block_mesh(*quad))
        .collect()
}
```

---

## Rule: `mc-perf-redstone-bfs` — BFS dirty-flag queue for redstone

Never scan the entire chunk for redstone state — only propagate changes.

```rust
use std::collections::VecDeque;
use ahash::AHashSet;

pub struct RedstoneGraph {
    /// Blocks waiting for update THIS tick
    pending: VecDeque<BlockPos>,
    /// Prevent duplicate enqueue within same wave
    enqueued: AHashSet<BlockPos>,
    /// Adjacency cache: when block A changes, which blocks need updating?
    /// Avoids recomputing block neighbour geometry every tick
    dependents: AHashMap<BlockPos, SmallVec<[BlockPos; 8]>>,
}

impl RedstoneGraph {
    pub fn tick(&mut self, world: &mut World) {
        // Process only current wave — new changes go to NEXT tick
        let wave: Vec<BlockPos> = self.pending.drain(..).collect();
        self.enqueued.clear();

        for pos in wave {
            let old_power = world.get_power(pos);
            let new_power = compute_redstone_power(world, pos);

            if old_power != new_power {
                world.set_power(pos, new_power);

                // Wake dependents — order matters (quasi connectivity!)
                if let Some(deps) = self.dependents.get(&pos) {
                    for &dep in deps {
                        if self.enqueued.insert(dep) {
                            self.pending.push_back(dep);
                        }
                    }
                }
            }
        }
    }

    /// Register that block `source` changing should wake `dependent`
    pub fn add_dependency(&mut self, source: BlockPos, dependent: BlockPos) {
        self.dependents.entry(source).or_default().push(dependent);
    }
}
```

---

## Rule: `mc-perf-parallel-gen` — Parallel chunk generation with rayon

Chunk base terrain generation is embarrassingly parallel (no deps between chunks).

```rust
use rayon::prelude::*;

pub fn generate_region(
    generator: &TerrainGenerator,
    positions: &[ChunkPos],
) -> AHashMap<ChunkPos, Chunk> {
    positions
        .par_iter()                          // rayon: uses all CPU cores
        .map(|&pos| (pos, generator.generate_chunk(pos)))
        .collect()
}

// Phase 2: decoration (trees, ores, structures) — needs neighbours, sequential
pub fn decorate_region(
    generator: &TerrainGenerator,
    chunks: &mut AHashMap<ChunkPos, Chunk>,
    inner_positions: &[ChunkPos],  // only positions with all 8 neighbours loaded
) {
    for &pos in inner_positions {
        generator.decorate_chunk(pos, chunks);
    }
}
```

---

## Rule: `mc-perf-spatial-entity-index` — Spatial index for entity queries

Avoid O(n) entity scans for nearby-entity queries:

```rust
/// Maps chunk position → entity IDs in that chunk.
/// Updated when entities cross chunk boundaries.
pub struct EntitySpatialIndex {
    by_chunk: AHashMap<ChunkPos, SmallVec<[EntityId; 8]>>,
}

impl EntitySpatialIndex {
    /// Entities near a position within radius (for mob AI, collision)
    pub fn entities_near(&self, pos: DVec3, radius: f64) -> impl Iterator<Item=EntityId> + '_ {
        let chunk_radius = (radius / 16.0).ceil() as i32;
        let center = ChunkPos::from_block(BlockPos {
            x: pos.x as i32, y: pos.y as i32, z: pos.z as i32
        });

        (-chunk_radius..=chunk_radius).flat_map(move |dx| {
            (-chunk_radius..=chunk_radius).flat_map(move |dz| {
                let query_pos = ChunkPos { x: center.x + dx, z: center.z + dz };
                self.by_chunk.get(&query_pos)
                    .into_iter()
                    .flat_map(|ids| ids.iter().copied())
            })
        })
    }
}
```

---

## Performance Benchmarks — Targets

| System | Target | Warning threshold | Action if exceeded |
|---|---|---|---|
| Entity physics (1000 entities) | < 5 ms | > 8 ms | Reduce physics granularity |
| Chunk generation (1 chunk) | < 20 ms | > 40 ms | Enable SIMD noise, parallelize |
| Chunk meshing (1 section) | < 2 ms | > 5 ms | Enable greedy meshing |
| Redstone tick (100-wire circuit) | < 1 ms | > 3 ms | Enable BFS dirty-flag |
| NBT read (1 chunk from disk) | < 5 ms | > 15 ms | Use memory mapping |
| Full server tick (100 players) | < 45 ms | > 50 ms | Profile immediately |

---

## Recommended Release Profile

```toml
[profile.release]
opt-level = 3
lto = "fat"         # Cross-crate inlining — critical for simdnoise, fastnbt
codegen-units = 1   # Single codegen unit — slower compile, faster runtime
panic = "abort"     # No unwinding — saves ~5% overhead in tight loops
strip = false        # Keep symbols for profiling

[profile.dev.package."*"]
opt-level = 3       # Optimize dependencies in dev (simdnoise is unusable at opt-0)
```

> **leonardomso rules**: `opt-lto-release`, `opt-codegen-units`, `perf-profile-first`.
