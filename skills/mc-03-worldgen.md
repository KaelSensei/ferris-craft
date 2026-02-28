---
name: mc-03-worldgen
description: >
  Layer 2 — Design Choices (WHAT). Procedural world generation in Rust: noise chains,
  density functions, biome blending, structure placement, and ore distribution.
  Mirrors Minecraft 1.18+ terrain generation model. Use when building a terrain
  generator, biome system, or any procedural world creation pipeline.
layer: 2
triggers:
  - worldgen, world gen, terrain, noise, perlin, simplex, biome, density function
  - cave gen, ore distribution, structure placement, seed, heightmap
---

# mc-03 — World Generation

> **Core Question**: *What generation model produces correct-looking, deterministic,
> performance-efficient terrain?*
>
> Minecraft 1.18+ uses a **multi-octave 3D density function** chain — not a heightmap.
> This is the gold standard for Minecraft-like world gen.

---

## Generation Pipeline (1.18+ style)

```
Seed (u64)
    │
    ├── 2D noise layers (XZ only — cheap, drives large-scale shape)
    │   ├── Continentalness   : land vs ocean distribution
    │   ├── Erosion           : how worn-down the terrain is
    │   ├── Peaks & Valleys   : local ruggedness
    │   └── Weirdness         : biome variant axis (selects biome in table)
    │
    ├── Spline interpolation  → initial_density(x, z)
    │   (continentalness × erosion × peaks_valleys → terrain height profile)
    │
    ├── 3D noise layer (XYZ — expensive, drives caves/overhangs)
    │   └── Cave density noise
    │
    └── Final density at (x, y, z):
        density = initial_density(x, z)
                + cave_noise(x, y, z) × 0.5
                - (y / 128.0)          ← vertical gradient
        if density > 0 → solid block
        if density ≤ 0 → air (or liquid below sea level)
```

---

## Rule: `mc-gen-simd-batch` — Always batch noise generation per section

Never call noise per-block in a loop. SIMD processes 4–8 values in parallel.

```rust
use simdnoise::NoiseBuilder;

pub struct TerrainGenerator {
    seed: i32,
}

impl TerrainGenerator {
    /// Generate density values for an entire 16×16×16 section in one SIMD call.
    /// Returns 4096 density values: index = y*256 + z*16 + x
    pub fn generate_section_density(
        &self,
        chunk_x: i32,
        section_y: i8,
        chunk_z: i32,
    ) -> Box<[f32; 4096]> {
        // simdnoise: processes 4 or 8 values per CPU cycle with AVX/SSE
        // 1 call here vs 4096 scalar calls — 4–8× throughput improvement
        let (density, _min, _max) = NoiseBuilder::gradient_3d_offset(
            chunk_x as f32 * 16.0,    16,   // X: start, count
            section_y as f32 * 16.0,  16,   // Y: start, count
            chunk_z as f32 * 16.0,    16,   // Z: start, count
        )
        .with_seed(self.seed)
        .with_octaves(4)
        .with_freq(0.005)
        .with_gain(0.5)
        .with_lacunarity(2.0)
        .generate();

        // density is Vec<f32> length 4096 — convert to Box<[f32; 4096]>
        density.try_into().unwrap_or_else(|_| unreachable!())
    }

    /// Generate full chunk column using section batching + rayon parallelism
    pub fn generate_chunk(&self, pos: ChunkPos) -> Chunk {
        use rayon::prelude::*;

        // Generate 24 sections in parallel — embarrassingly parallel, no deps
        let sections: Vec<ChunkSection> = (-4_i8..=19_i8)
            .into_par_iter()
            .map(|section_y| {
                let density = self.generate_section_density(pos.x, section_y, pos.z);
                self.density_to_blocks(&density, section_y)
            })
            .collect();

        Chunk { sections: sections.into_boxed_slice().try_into().unwrap(), pos }
    }

    fn density_to_blocks(&self, density: &[f32; 4096], section_y: i8) -> ChunkSection {
        let mut section = ChunkSection::new();
        for index in 0..4096 {
            let block = if density[index] > 0.0 {
                let (x, y, z) = index_to_xyz(index);
                let world_y = section_y as i32 * 16 + y as i32;
                self.select_block(world_y)
            } else {
                let (_, y, _) = index_to_xyz(index);
                let world_y = section_y as i32 * 16 + y as i32;
                if world_y <= 62 { WATER } else { AIR }  // sea level = 62
            };
            section.blocks.set(index, block);
        }
        section
    }

    fn select_block(&self, world_y: i32) -> BlockState {
        match world_y {
            i32::MIN..=0  => DEEPSLATE,
            1..=58        => STONE,
            59..=62       => DIRT,
            63            => GRASS_BLOCK,
            _             => STONE,  // mountain peaks etc.
        }
    }
}
```

---

## Rule: `mc-gen-2d-noise` — Separate 2D and 3D noise layers

2D noise is ~16× cheaper than 3D per chunk column. Compute it once, reuse for all Y.

```rust
pub struct TerrainNoise {
    seed: i32,
    /// 2D layers — computed once per XZ, reused for all 24 sections
    continents:    OctaveNoise2D,
    erosion:       OctaveNoise2D,
    peaks_valleys: OctaveNoise2D,
    weirdness:     OctaveNoise2D,
    /// 3D layer — computed per section (expensive, only for caves)
    cave_density:  OctaveNoise3D,
}

impl TerrainNoise {
    /// Compute 2D terrain parameters for a full 16×16 column (256 XZ points).
    /// Returns per-XZ: (continentalness, erosion, peaks_valleys, weirdness)
    pub fn compute_column_params(&self, chunk_x: i32, chunk_z: i32)
        -> Box<[[f32; 4]; 256]>
    {
        // One SIMD batch for each 2D noise layer: 4 calls × 256 values
        let c  = self.continents.sample_chunk(chunk_x, chunk_z);
        let e  = self.erosion.sample_chunk(chunk_x, chunk_z);
        let pv = self.peaks_valleys.sample_chunk(chunk_x, chunk_z);
        let w  = self.weirdness.sample_chunk(chunk_x, chunk_z);

        let mut result = Box::new([[0f32; 4]; 256]);
        for i in 0..256 {
            result[i] = [c[i], e[i], pv[i], w[i]];
        }
        result
    }

    /// Final density at a specific (x, y, z) world position.
    /// Uses precomputed column params to avoid recomputing 2D noise per block.
    pub fn density_at(
        &self,
        column_params: &[[f32; 4]; 256],
        local_xz_index: usize,  // 0..256 = z*16+x
        world_y: f32,
        cave_noise: f32,        // precomputed from 3D SIMD batch
    ) -> f32 {
        let [c, e, pv, _w] = column_params[local_xz_index];

        // Spline lookup — replaces Minecraft's internal spline system
        let initial_density = terrain_spline(c, e, pv);

        // Combine: terrain shape + cave carving + vertical falloff
        initial_density
            + cave_noise * 0.3
            - (world_y / 128.0)    // positive Y → less likely to be solid
    }
}

/// Simple terrain spline approximation.
/// Real Minecraft uses a lookup table from a trained spline — replace as needed.
fn terrain_spline(continentalness: f32, erosion: f32, peaks_valleys: f32) -> f32 {
    let base_height = continentalness * 64.0 - 32.0;  // −32 to +32 blocks from sea
    let roughness   = (1.0 - erosion) * peaks_valleys * 32.0;
    (base_height + roughness) / 128.0  // normalize to density-function scale
}
```

---

## Rule: `mc-gen-deterministic` — Enforce generation determinism

World generation must produce **identical output for the same seed** across platforms.
Float non-determinism (different SIMD backends, compiler opts) can break this.

```rust
// ❌ Risk: platform-dependent float results
fn noisy_value(x: f64, y: f64) -> f64 {
    (x * 0.1).sin() * (y * 0.1).cos()  // transcendentals vary by platform
}

// ✅ Use a fixed-seed, deterministic noise library
// simdnoise: same seed → same output on all x86_64 platforms with same feature set
// noise (pure Rust): fully deterministic, no SIMD

// For cross-platform bit-exact determinism: use `noise` crate, not `simdnoise`
// For same-platform performance (server + client same binary): `simdnoise` is fine

use noise::{NoiseFn, SuperSimplex};

pub struct DeterministicGenerator {
    noise: SuperSimplex,  // cryptographically seeded, deterministic
}

impl DeterministicGenerator {
    pub fn new(seed: u32) -> Self {
        Self { noise: SuperSimplex::new(seed) }
    }

    pub fn height_at(&self, x: i32, z: i32) -> i32 {
        let n = self.noise.get([x as f64 * 0.01, z as f64 * 0.01]);
        (n * 32.0 + 64.0) as i32  // Y range: 32..96
    }
}
```

---

## Rule: `mc-gen-decoration-two-phase` — Two-phase generation for structures

Structures (trees, villages, dungeons) span chunk boundaries. Generating them in one
pass causes corruption. Use two explicit phases:

```rust
pub enum GenerationPhase {
    /// Phase 1: base terrain only, no cross-chunk dependencies
    /// Safe to parallelize with rayon::par_iter
    BaseTerrain,

    /// Phase 2: decoration (trees, ores, structures)
    /// Requires N neighbours to be in Phase 1+ first
    Decoration { neighbours_ready: bool },
}

impl ChunkGenerator {
    pub fn generate_all(&self, positions: &[ChunkPos]) -> HashMap<ChunkPos, Chunk> {
        // Phase 1: parallel — no dependencies
        let mut chunks: HashMap<_, _> = positions
            .par_iter()
            .map(|&pos| (pos, self.generate_base_terrain(pos)))
            .collect();

        // Phase 2: sequential — needs neighbours from Phase 1
        // Only decorate chunks whose 3×3 neighbourhood is fully in Phase 1
        for &pos in positions {
            let all_neighbours_ready = (-1..=1).all(|dx| (-1..=1).all(|dz| {
                let neighbour = ChunkPos { x: pos.x + dx, z: pos.z + dz };
                chunks.contains_key(&neighbour)
            }));

            if all_neighbours_ready {
                let decoration = self.generate_decoration(pos, &chunks);
                self.apply_decoration(chunks.get_mut(&pos).unwrap(), decoration);
            }
        }

        chunks
    }
}
```

---

## Noise Algorithm Selection

| Algorithm | Crate | Use case | Notes |
|---|---|---|---|
| OpenSimplex2 | `noise` | Default terrain | Smooth, no directional artifacts |
| SuperSimplex | `noise` | Biome blend | Better gradients than simplex |
| Perlin | `simdnoise` | Fast bulk generation | SIMD — 4–8× speedup |
| Fractal Brownian Motion | `simdnoise` | Heightmaps, caves | Multi-octave, configurable |
| Worley (Cellular) | `noise` | Cave shapes, cracks | Good for irregular features |
| Value noise | `noise` | Ore patches | Cheaper, blockier appearance |
