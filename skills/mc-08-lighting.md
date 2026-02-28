---
name: mc-08-lighting
description: >
  Layer 2 — Design Choices (WHAT). Minecraft-accurate lighting engine in Rust:
  heightmap-based skylight initialisation, BFS flood-fill for block and sky light,
  cross-chunk border propagation, and light removal (the hard part). Based on the
  Starlight/Phosphor analysis and the official Minecraft wiki light specification.
  Use when implementing chunk lighting, handling block placement/removal updates,
  or debugging light leaks at chunk borders.
layer: 2
triggers:
  - lighting, light engine, skylight, block light, light level, flood fill
  - light propagation, light update, heightmap, torch light, ambient occlusion
  - light leak, dark chunk, chunk border light, NibbleArray lighting
---

# mc-08 — Lighting Engine

> **Core Question**: *How does Minecraft propagate 0–15 light values across
> 30 million × 30 million × 384 blocks without recomputing everything on every change?*
>
> Answer: **BFS flood-fill with dirty-flag queues** — one queue per light type,
> separate addition and removal passes.

---

## Key Facts (from Minecraft Wiki + Starlight analysis)

| Property | Value |
|---|---|
| Light range | 0–15 (4 bits per block — stored in `NibbleArray`) |
| Channels | 2 — **sky light** (sun) and **block light** (torches etc.) |
| Sky light at surface | Always 15 for blocks directly exposed to sky |
| Block light from torch | 14; decreases by 1 per taxicab-distance step |
| Propagation rule | Each step to adjacent block: `new_level = source_level − max(1, opacity)` |
| Opaque block opacity | 15 (fully blocks light) |
| Glass / slabs opacity | 0 (transparent) or directional |
| Night cycle | Does NOT change stored sky light — mob spawning reads a time-adjusted value |
| Starlight insight | One unified propagator for both channels, parameterised on source setup |

---

## Architecture: Two Passes, Two Queues

```
Block placed / removed
        │
        ▼
┌───────────────────┐     ┌─────────────────────────┐
│  REMOVAL PASS     │────▶│  ADD PASS (BFS)          │
│  (BFS, harder)    │     │  Re-add all sources that │
│  Zero out stale   │     │  were touching the area  │
│  light values     │     └─────────────────────────┘
└───────────────────┘

Chunk first load:
        │
        ▼
┌───────────────────┐
│  HEIGHTMAP INIT   │  Build per-column max-opaque-Y heightmap
│  (cheap, 2D)      │
└────────┬──────────┘
         ▼
┌───────────────────┐
│  SKY INIT PASS    │  Inject level-15 sources from heightmap tops
│  (BFS downward)   │  into the main propagation queue
└────────┬──────────┘
         ▼
┌───────────────────┐
│  BLOCK INIT PASS  │  Scan palette for emitting blocks
│  (palette-opt)    │  (only check non-air sections)
└────────┬──────────┘
         ▼
┌───────────────────┐
│  PROPAGATION BFS  │  Process combined queue until empty
│  (main algorithm) │
└───────────────────┘
```

---

## Core Data Structures

```rust
use std::collections::VecDeque;
use ahash::AHashMap;

/// One light update queued for BFS processing.
#[derive(Clone, Copy)]
struct LightNode {
    pos: BlockPos,
    level: u8,  // 0–15
}

/// The full lighting state for a loaded world.
pub struct LightEngine {
    /// Sky light nibble arrays per chunk section
    sky_light: AHashMap<(i32, i8, i32), Box<NibbleArray>>,  // (cx, cy, cz) → array
    /// Block light nibble arrays per chunk section
    block_light: AHashMap<(i32, i8, i32), Box<NibbleArray>>,
    /// Per-column heightmap: highest Y where opacity > 0
    heightmap: AHashMap<(i32, i32), HeightMap>,  // (cx, cz) → 16×16 i32 array
}

/// 16×16 array of max-opaque-Y values for one chunk column.
/// heightmap[x + z*16] = world Y of highest block with opacity > 0
pub struct HeightMap(pub [i32; 256]);

impl HeightMap {
    /// Sky is exposed at this XZ column if world_y > heightmap value
    pub fn is_sky_exposed(&self, local_x: u8, local_z: u8, world_y: i32) -> bool {
        world_y > self.0[local_x as usize + local_z as usize * 16]
    }
}
```

---

## Rule: `mc-light-heightmap` — Build heightmap first, then skylight

Building the heightmap once per chunk is far cheaper than scanning top-to-bottom
for every XZ column individually.

```rust
impl LightEngine {
    /// Build the heightmap for a loaded chunk column.
    /// Must be called BEFORE any sky light propagation for this chunk.
    pub fn build_heightmap(&mut self, chunk: &Chunk, cx: i32, cz: i32) {
        let mut hmap = HeightMap([i32::MIN; 256]);

        // Iterate sections top-down — stop early once we find opaque blocks
        for section_idx in (0..24usize).rev() {
            let section_y = section_idx as i8 - 4;  // sections start at Y=−64 (index 0)
            let world_y_base = section_y as i32 * 16;

            let section = &chunk.sections[section_idx];

            // Palette optimisation (Starlight approach):
            // If the entire section is air (Single(AIR)), skip it entirely
            if section.blocks.is_all_air() { continue; }

            for local_z in 0u8..16 {
                for local_x in 0u8..16 {
                    let col_idx = local_x as usize + local_z as usize * 16;
                    if hmap.0[col_idx] != i32::MIN { continue; }  // already found top

                    // Scan Y from top of section downward
                    for local_y in (0u8..16).rev() {
                        let block = section.blocks.get(section_index(local_x, local_y, local_z));
                        if block_opacity(block) > 0 {
                            hmap.0[col_idx] = world_y_base + local_y as i32;
                            break;
                        }
                    }
                }
            }
        }

        self.heightmap.insert((cx, cz), hmap);
    }
}
```

---

## Rule: `mc-light-bfs` — Unified BFS propagator

Starlight's key insight: **one propagation algorithm** handles both sky and block
light. The only difference is how sources are initially seeded into the queue.

```rust
impl LightEngine {
    /// Core BFS propagation — add light outward from all queued sources.
    /// Works for both sky light and block light depending on which NibbleArray
    /// map is passed in.
    fn propagate_add(
        &mut self,
        light_map: &mut AHashMap<(i32, i8, i32), Box<NibbleArray>>,
        seed_queue: VecDeque<LightNode>,
        world: &World,
    ) {
        let mut queue = seed_queue;

        while let Some(node) = queue.pop_front() {
            if node.level == 0 { continue; }

            let current = self.get_light(light_map, node.pos);
            if current >= node.level { continue; }  // already brighter, skip

            self.set_light(light_map, node.pos, node.level);

            // Propagate to all 6 neighbours
            for dir in Direction::ALL {
                let neighbour = node.pos.offset(dir);
                let opacity = world.block_opacity_in_direction(neighbour, dir.opposite());
                let new_level = node.level.saturating_sub(opacity.max(1));

                if new_level > self.get_light(light_map, neighbour) {
                    queue.push_back(LightNode { pos: neighbour, level: new_level });
                }
            }
        }
    }

    /// Seed sky light from the heightmap into the BFS queue.
    /// Called once per chunk load, after build_heightmap.
    pub fn init_sky_light(&mut self, chunk: &Chunk, cx: i32, cz: i32, world: &World) {
        let hmap = &self.heightmap[&(cx, cz)];
        let mut queue = VecDeque::new();

        for local_z in 0u8..16 {
            for local_x in 0u8..16 {
                let col_top = hmap.0[local_x as usize + local_z as usize * 16];

                // Inject level-15 sky light at every Y above the heightmap
                // (directly sky-exposed blocks)
                let world_x = cx * 16 + local_x as i32;
                let world_z = cz * 16 + local_z as i32;

                let sky_top = 319i32;  // 1.18+ build limit top
                for world_y in (col_top + 1)..=sky_top {
                    queue.push_back(LightNode {
                        pos: BlockPos { x: world_x, y: world_y, z: world_z },
                        level: 15,
                    });
                }

                // Also consider sideways propagation from neighbouring columns
                // whose heightmap is lower (sunlight bends around the corner)
                // → handled automatically by BFS propagating sideways
            }
        }

        self.propagate_add(&mut self.sky_light, queue, world);
    }

    /// Seed block light from all emitting blocks in a chunk section.
    /// Uses the palette to avoid iterating all 4096 blocks when section is all-air.
    pub fn init_block_light(&mut self, chunk: &Chunk, cx: i32, section_idx: usize,
                             cz: i32, world: &World)
    {
        let section = &chunk.sections[section_idx];
        if section.blocks.is_all_air() { return; }  // nothing to emit

        let section_y = section_idx as i8 - 4;
        let world_y_base = section_y as i32 * 16;
        let mut queue = VecDeque::new();

        // Palette scan: only check block types that can emit light
        // This is the Starlight / Phosphor optimisation — avoids 4096 lookups
        // when the section's palette contains no emitting blocks
        let has_emitters = section.blocks.palette_any(|b| light_emission(b) > 0);
        if !has_emitters { return; }

        for local_y in 0u8..16 {
            for local_z in 0u8..16 {
                for local_x in 0u8..16 {
                    let block = section.blocks.get(section_index(local_x, local_y, local_z));
                    let emission = light_emission(block);
                    if emission > 0 {
                        queue.push_back(LightNode {
                            pos: BlockPos {
                                x: cx * 16 + local_x as i32,
                                y: world_y_base + local_y as i32,
                                z: cz * 16 + local_z as i32,
                            },
                            level: emission,
                        });
                    }
                }
            }
        }

        self.propagate_add(&mut self.block_light, queue, world);
    }
}
```

---

## Rule: `mc-light-removal` — Light removal is the hard part

Adding light is a straightforward BFS. **Removing light requires a two-phase approach**:
first zero out stale values (tracking the old level), then re-add from surviving sources.

```rust
impl LightEngine {
    /// Handle a block placement that may reduce light in the area.
    pub fn on_block_placed(
        &mut self,
        pos: BlockPos,
        new_block: BlockState,
        world: &World,
    ) {
        // Phase 1: Remove stale light values outward from the changed block
        let old_sky   = self.get_light(&self.sky_light.clone(), pos);
        let old_block = self.get_light(&self.block_light.clone(), pos);

        self.remove_light(&mut self.sky_light.clone(),   pos, old_sky,   world);
        self.remove_light(&mut self.block_light.clone(), pos, old_block, world);

        // Phase 2: Re-seed from any surviving sources touching the darkened area
        // Sky: re-check heightmap column
        self.reinit_sky_column(pos.x, pos.z, world);

        // Block: re-seed from emission of the new block itself, if any
        let emission = light_emission(new_block);
        if emission > 0 {
            let mut queue = VecDeque::new();
            queue.push_back(LightNode { pos, level: emission });
            self.propagate_add(&mut self.block_light, queue, world);
        }
    }

    /// BFS removal pass: zero out light values, collecting re-seed candidates.
    fn remove_light(
        &mut self,
        light_map: &mut AHashMap<(i32, i8, i32), Box<NibbleArray>>,
        start: BlockPos,
        start_level: u8,
        world: &World,
    ) {
        // removal_queue: (pos, level_that_was_here)
        let mut removal_queue: VecDeque<(BlockPos, u8)> = VecDeque::new();
        let mut re_seed: VecDeque<LightNode> = VecDeque::new();

        removal_queue.push_back((start, start_level));

        while let Some((pos, level)) = removal_queue.pop_front() {
            self.set_light(light_map, pos, 0);

            for dir in Direction::ALL {
                let nb = pos.offset(dir);
                let nb_level = self.get_light(light_map, nb);

                if nb_level == 0 { continue; }

                if nb_level < level {
                    // This neighbour was lit by our removed source → remove it too
                    removal_queue.push_back((nb, nb_level));
                } else {
                    // This neighbour has an independent source → re-seed from it
                    re_seed.push_back(LightNode { pos: nb, level: nb_level });
                }
            }
        }

        // Re-propagate from surviving independent sources
        self.propagate_add(light_map, re_seed, world);
    }
}
```

---

## Rule: `mc-light-cross-chunk` — Cross-chunk border propagation

When a chunk loads next to an already-lit chunk, border light must flow in.
This is where most "dark chunk" bugs come from.

```rust
impl LightEngine {
    /// After a chunk finishes loading, check all 4 neighbour chunks for
    /// border light that should spill INTO the new chunk.
    ///
    /// Based on Starlight's edge-check: does the light value at the border
    /// match what its neighbour says it should be? If not, propagate.
    pub fn propagate_chunk_borders(
        &mut self,
        cx: i32,
        cz: i32,
        world: &World,
    ) {
        // Check all 4 horizontal borders (N/S/E/W)
        let border_pairs = [
            // (neighbour chunk, direction INTO new chunk, x/z range of border column)
            (cx,   cz-1, Direction::North),
            (cx,   cz+1, Direction::South),
            (cx-1, cz,   Direction::West),
            (cx+1, cz,   Direction::East),
        ];

        for (ncx, ncz, dir) in border_pairs {
            if !world.is_chunk_loaded(ncx, ncz) { continue; }

            let mut sky_queue   = VecDeque::new();
            let mut block_queue = VecDeque::new();

            // Scan border column between neighbour and new chunk
            for section_y in -4i8..=19 {
                let world_y_base = section_y as i32 * 16;

                for local_a in 0u8..16 {
                    // border position in the neighbour chunk
                    let (nb_x, nb_z, new_x, new_z) = match dir {
                        Direction::North => (ncx*16 + local_a as i32, ncz*16+15,
                                            cx*16  + local_a as i32, cz*16),
                        Direction::South => (ncx*16 + local_a as i32, ncz*16,
                                            cx*16  + local_a as i32, cz*16+15),
                        Direction::West  => (ncx*16+15, ncz*16 + local_a as i32,
                                            cx*16,     cz*16  + local_a as i32),
                        Direction::East  => (ncx*16,   ncz*16 + local_a as i32,
                                            cx*16+15,  cz*16  + local_a as i32),
                    };

                    for local_y in 0u8..16 {
                        let world_y = world_y_base + local_y as i32;
                        let nb_pos  = BlockPos { x: nb_x,  y: world_y, z: nb_z };
                        let new_pos = BlockPos { x: new_x, y: world_y, z: new_z };

                        let opacity = world.block_opacity_in_direction(new_pos, dir.opposite());

                        let nb_sky = self.get_light(&self.sky_light, nb_pos);
                        if nb_sky.saturating_sub(opacity.max(1))
                            > self.get_light(&self.sky_light, new_pos)
                        {
                            sky_queue.push_back(LightNode {
                                pos: new_pos,
                                level: nb_sky.saturating_sub(opacity.max(1)),
                            });
                        }

                        let nb_block = self.get_light(&self.block_light, nb_pos);
                        if nb_block.saturating_sub(opacity.max(1))
                            > self.get_light(&self.block_light, new_pos)
                        {
                            block_queue.push_back(LightNode {
                                pos: new_pos,
                                level: nb_block.saturating_sub(opacity.max(1)),
                            });
                        }
                    }
                }
            }

            self.propagate_add(&mut self.sky_light,   sky_queue,   world);
            self.propagate_add(&mut self.block_light, block_queue, world);
        }
    }
}
```

---

## Block Light Emission Reference

```rust
/// Returns the light emission level of a block (0 = no emission).
pub fn light_emission(block: BlockState) -> u8 {
    match block.id() {
        // Full brightness
        id if is_glowstone(id)    => 15,
        id if is_shroomlight(id)  => 15,
        id if is_beacon(id)       => 15,
        // High brightness
        id if is_torch(id)        => 14,
        id if is_lava(id)         => 15,
        id if is_fire(id)         => 15,
        id if is_jack_o_lantern(id) => 15,
        // Medium
        id if is_lantern(id)      => 15,
        id if is_sea_lantern(id)  => 15,
        id if is_sea_pickle(id)   => 6,   // varies by count
        id if is_soul_torch(id)   => 10,
        // Redstone
        id if is_redstone_lamp_on(id) => 15,
        id if is_redstone_torch(id)   => 7,
        _ => 0,
    }
}

/// Returns the opacity of a block (0 = transparent, 15 = fully opaque).
pub fn block_opacity(block: BlockState) -> u8 {
    match block.id() {
        id if is_air(id)          => 0,
        id if is_glass(id)        => 0,
        id if is_leaves(id)       => 1,   // leaves reduce by 1
        id if is_water(id)        => 1,   // water reduces by 1
        id if is_ice(id)          => 0,
        id if is_slabs(id)        => 0,   // directional — simplified here
        _                         => 15,  // solid opaque block
    }
}
```

---

## Performance Notes

| Optimisation | Source | Impact |
|---|---|---|
| Palette scan before section scan | Starlight | Skip entirely-air sections in block light init |
| Heightmap for sky initialisation | Starlight | O(16×16) per chunk vs O(16×16×384) |
| Two-pass removal (not scan-all) | Phosphor | Skip unrelated chunks during source removal |
| Defer lighting until neighbour queried | Phosphor | Batch updates, reduce duplicate scheduled work |
| Separate sky + block queues, parallel | Starlight | Independent channels can run concurrently |

> **Sources**: [Minecraft Wiki — Light](https://minecraft.wiki/w/Light) ·
> [Starlight TECHNICAL_DETAILS.md](https://github.com/PaperMC/Starlight/blob/fabric/TECHNICAL_DETAILS.md) ·
> [dktapps lighting-algorithm-spec](https://github.com/dktapps/lighting-algorithm-spec) ·
> [0fps.net — Voxel Lighting](https://0fps.net/2018/02/21/voxel-lighting/)
