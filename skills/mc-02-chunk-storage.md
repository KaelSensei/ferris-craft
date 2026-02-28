---
name: mc-02-chunk-storage
description: >
  Layer 2 — Design Choices (WHAT). Chunk data layout, paletted containers, section
  indexing, and memory-efficient storage. Use when designing or debugging chunk data
  structures, block state storage, biome arrays, or light data.
layer: 2
triggers:
  - chunk, section, palette, block state storage, chunk layout, block array, biome array
---

# mc-02 — Chunk Storage Design

> **Core Question**: *What is the most cache-friendly, memory-efficient layout for
> 16×16×384 blocks?*
>
> Answer: **Paletted section arrays** — not HashMaps, not flat `u16` arrays without palette.

---

## Architecture Overview

```
Chunk Column (16 × 16 × 384)
│
├── Section[0]  Y: −64 to −49   (16×16×16 = 4096 blocks)
├── Section[1]  Y: −48 to −33
│   ...
├── Section[14] Y: 144 to 159
│   ...
└── Section[23] Y: 304 to 319   ← build limit top

Each ChunkSection contains:
  ├── PalettedContainer<BlockState, 4096>  (block states)
  ├── PalettedContainer<BiomeId, 64>       (4×4×4 biome grid)
  ├── NibbleArray  sky_light  [2048 bytes]  (optional — sky-exposed)
  └── NibbleArray  block_light [2048 bytes]
```

---

## Rule: `mc-chunk-packed-array` — Use packed section arrays, not HashMaps

```rust
// ❌ Wrong: O(1) lookup but terrible cache performance under chunk iteration
struct BadChunk {
    blocks: HashMap<(i32, i32, i32), BlockState>,  // 98,304 hash lookups per render
}

// ✅ Correct: packed sections with palette compression
pub struct Chunk {
    /// 24 sections for 1.18+ build height (−64 to +320)
    sections: Box<[ChunkSection; 24]>,
    /// Global chunk metadata
    pub pos: ChunkPos,
    pub inhabited_time: u64,
}

pub struct ChunkSection {
    blocks: PalettedContainer<BlockState, 4096>,
    biomes: PalettedContainer<BiomeId, 64>,
    sky_light: Option<Box<NibbleArray>>,   // None if section fully in darkness
    block_light: Box<NibbleArray>,
    /// Cache: how many non-air blocks (avoids iterating to check emptiness)
    non_air_count: u16,
}
```

---

## Rule: `mc-chunk-index` — Y-major index for cache-friendly skylight propagation

Minecraft iterates in the Y direction for skylight propagation.
Use Y-major (Y × 256 + Z × 16 + X) for best cache behavior:

```rust
/// Convert local (x, y, z) within a section to a flat array index.
/// x, y, z are all in range 0..16.
///
/// Y-major layout: skylight propagation iterates top-down (Y axis),
/// so Y-major gives sequential access during light calculation.
#[inline(always)]
pub fn section_index(x: u8, y: u8, z: u8) -> usize {
    debug_assert!(x < 16 && y < 16 && z < 16);
    (y as usize) << 8 | (z as usize) << 4 | x as usize
    //  Y * 256        Z * 16              X
}

/// Reverse: flat index → local (x, y, z)
#[inline(always)]
pub fn index_to_xyz(index: usize) -> (u8, u8, u8) {
    let x = (index & 0xF) as u8;
    let z = ((index >> 4) & 0xF) as u8;
    let y = ((index >> 8) & 0xF) as u8;
    (x, y, z)
}
```

---

## Rule: `mc-chunk-palette` — Implement paletted containers correctly

Minecraft's protocol spec mandates 3 palette modes based on unique block count.
This is critical for protocol-compatible servers:

```rust
use smallvec::SmallVec;
use bitvec::vec::BitVec;

/// Palette compression — matches Minecraft protocol specification.
///
/// Single:   0–1 unique values → no bit array, just one value
/// Indirect: 2–16 unique values → 4-bit indices into local palette
/// Direct:   >16 unique values → global palette IDs, 15 bits per entry
pub enum PalettedContainer<T: Copy + Default + Eq, const N: usize> {
    /// All blocks in this section are the same (e.g., all air)
    Single(T),

    /// Local palette — efficient for low-variety sections (most are!)
    Indirect {
        /// SmallVec: palette rarely exceeds 16 entries — stays on stack
        palette: SmallVec<[T; 16]>,
        /// Bit-packed indices — bits_per_entry = ceil(log2(palette.len()))
        data: BitVec,
    },

    /// Direct global IDs — for sections with >16 unique block types
    Direct {
        data: BitVec,  // 15 bits per entry (block), 6 bits per entry (biome)
    },
}

impl<T: Copy + Default + Eq, const N: usize> PalettedContainer<T, N> {
    pub fn new() -> Self {
        Self::Single(T::default())
    }

    pub fn get(&self, index: usize) -> T {
        match self {
            Self::Single(v) => *v,
            Self::Indirect { palette, data } => {
                let bpe = usize::BITS as usize
                    - palette.len().next_power_of_two().leading_zeros() as usize;
                let bpe = bpe.max(4); // minimum 4 bits per entry
                let raw = get_bits(&data, index * bpe, bpe) as usize;
                palette[raw]
            }
            Self::Direct { data } => {
                let bpe = 15; // global block palette always 15 bits
                let raw = get_bits(&data, index * bpe, bpe) as usize;
                // Caller must map raw ID to T via global registry
                todo!("implement global registry lookup")
            }
        }
    }

    pub fn set(&mut self, index: usize, value: T) {
        match self {
            Self::Single(v) if *v == value => {}  // no-op, already this value
            Self::Single(existing) => {
                // Upgrade Single → Indirect
                let mut palette = SmallVec::new();
                palette.push(*existing);
                palette.push(value);
                let mut data = BitVec::with_capacity(N * 4);
                data.resize(N * 4, false);
                // All existing blocks were `existing` → all indices = 0
                // Set index to 1 (new value)
                set_bits(&mut data, index * 4, 4, 1);
                *self = Self::Indirect { palette, data };
            }
            Self::Indirect { palette, data } => {
                let pos = palette.iter().position(|&v| v == value)
                    .unwrap_or_else(|| {
                        palette.push(value);
                        palette.len() - 1
                    });
                // Upgrade to Direct if palette exceeds indirect threshold
                if palette.len() > 16 {
                    // TODO: upgrade to Direct
                }
                let bpe = /* recalculate */ 4usize; // simplified
                set_bits(data, index * bpe, bpe, pos as u64);
            }
            Self::Direct { data } => {
                set_bits(data, index * 15, 15, value as u64); // requires T: Into<u64>
            }
        }
    }
}

fn get_bits(data: &BitVec, bit_start: usize, bit_len: usize) -> u64 {
    (0..bit_len).fold(0u64, |acc, i| {
        acc | ((*data.get(bit_start + i).unwrap_or(&false) as u64) << i)
    })
}

fn set_bits(data: &mut BitVec, bit_start: usize, bit_len: usize, value: u64) {
    for i in 0..bit_len {
        data.set(bit_start + i, (value >> i) & 1 == 1);
    }
}
```

---

## Rule: `mc-nibble-array` — 4-bit packed arrays for light data

Light values are 0–15: 4 bits. Storing as `u8` wastes 50% memory.
Chunks are sent to every player — light data compactness matters.

```rust
/// 4-bit packed nibble array for block/sky light storage.
/// 4096 light values stored in 2048 bytes.
pub struct NibbleArray(pub [u8; 2048]);

impl NibbleArray {
    pub fn new() -> Self { Self([0u8; 2048]) }

    /// Get light level for block at section-local index (0..4096)
    #[inline(always)]
    pub fn get(&self, index: usize) -> u8 {
        let byte = self.0[index >> 1];
        if index & 1 == 0 { byte & 0x0F } else { (byte >> 4) & 0x0F }
    }

    /// Set light level (value must be 0..=15)
    #[inline(always)]
    pub fn set(&mut self, index: usize, value: u8) {
        debug_assert!(value <= 15, "light value out of range 0..=15");
        let byte = &mut self.0[index >> 1];
        if index & 1 == 0 {
            *byte = (*byte & 0xF0) | (value & 0x0F);
        } else {
            *byte = (*byte & 0x0F) | ((value & 0x0F) << 4);
        }
    }

    /// Fill entire array with a single value (e.g., 15 for full sky light)
    pub fn fill(&mut self, value: u8) {
        debug_assert!(value <= 15);
        let byte = (value & 0x0F) | ((value & 0x0F) << 4);
        self.0.fill(byte);
    }
}
```

---

## Memory Budget Reference

| Data | Size per section | Per chunk column (24 sections) |
|---|---|---|
| Block states (single) | 2 bytes | 48 bytes |
| Block states (indirect, 8 unique) | ~2KB | ~48KB |
| Block states (direct) | ~7.7KB | ~185KB |
| Biome data | 64 × 4 bytes = 256 bytes | 6KB |
| Sky light (nibble) | 2KB | 48KB |
| Block light (nibble) | 2KB | 48KB |
| **Total (average loaded chunk)** | | **~150–200KB** |

> With 1000 loaded chunks (20×20 view distance + margin), expect **150–200MB** chunk RAM.
> This is why single-value palette optimization (most sections are all-air) is critical.
