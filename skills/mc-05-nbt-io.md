---
name: mc-05-nbt-io
description: >
  Layer 1 — Language Mechanics (HOW). Parsing NBT (Named Binary Tag) data and reading
  Minecraft Anvil region files (.mca). Covers fastnbt, flate2, memmap2, region file
  sector layout, and safe error handling for malformed world data.
layer: 1
triggers:
  - NBT, SNBT, .mca, region file, level.dat, Anvil format, fastnbt, chunk save, chunk load
---

# mc-05 — NBT & Region File I/O

> **Core Question**: *How to read and write Minecraft world data without allocating
> for every block or crashing on malformed files?*
>
> Key: memory-map the .mca file, read sector table, decompress with flate2,
> parse NBT with fastnbt. Never `read_to_end` the entire region file.

---

## Anvil Region Format Overview

```
region/r.0.0.mca  ← region at chunk coords 0–31, 0–31
region/r.-1.0.mca ← region at chunk coords −32..−1, 0–31

File layout:
  bytes 0–4095:   Location table  (1024 × 4-byte entries)
  bytes 4096–8191: Timestamp table (1024 × 4-byte entries)
  bytes 8192+:    Chunk data      (variable, 4096-byte sector-aligned)

Location entry (4 bytes):
  bits 31–8:  Sector offset  (multiply by 4096 to get byte offset)
  bits  7–0:  Sector count   (chunk data length in 4096-byte sectors)
  Value 0: chunk not generated

Chunk data block:
  bytes 0–3: Data length (big-endian u32)
  byte    4: Compression type (1=gzip, 2=zlib, 3=none, 4=lz4)
  bytes 5+:  Compressed NBT data
```

---

## Rule: `mc-nbt-region-mmap` — Memory-map region files for zero-copy reads

```rust
use memmap2::Mmap;
use std::fs::File;
use std::path::Path;

pub struct RegionFile {
    mmap: Mmap,  // zero-copy — OS pages in only what we access
}

impl RegionFile {
    pub fn open(path: &Path) -> std::io::Result<Self> {
        let file = File::open(path)?;
        // SAFETY: we only read this file; no concurrent writes (single-process)
        let mmap = unsafe { Mmap::map(&file)? };
        Ok(Self { mmap })
    }

    /// Read raw compressed NBT bytes for a chunk.
    /// Returns None if the chunk has not been generated yet.
    pub fn raw_chunk_data(&self, local_x: u8, local_z: u8)
        -> Option<(u8, &[u8])>  // (compression_type, compressed_data)
    {
        debug_assert!(local_x < 32 && local_z < 32);
        let location_index = (local_x as usize + local_z as usize * 32) * 4;

        // Read sector offset (3 bytes big-endian) + sector count (1 byte)
        let offset_be = [
            0,
            self.mmap[location_index],
            self.mmap[location_index + 1],
            self.mmap[location_index + 2],
        ];
        let sector_offset = u32::from_be_bytes(offset_be) as usize;
        let _sector_count = self.mmap[location_index + 3];

        if sector_offset == 0 { return None; }  // chunk not generated

        let byte_offset = sector_offset * 4096;

        // Read data length (4 bytes, big-endian)
        let length = u32::from_be_bytes([
            self.mmap[byte_offset],
            self.mmap[byte_offset + 1],
            self.mmap[byte_offset + 2],
            self.mmap[byte_offset + 3],
        ]) as usize;

        if length == 0 || byte_offset + 5 + length > self.mmap.len() {
            return None;  // corrupted entry — don't crash
        }

        let compression = self.mmap[byte_offset + 4];
        let data = &self.mmap[byte_offset + 5..byte_offset + 4 + length];

        Some((compression, data))
    }
}
```

---

## Rule: `mc-nbt-decompress-parse` — Decompress then parse NBT

```rust
use flate2::read::{GzDecoder, ZlibDecoder};
use fastnbt::Value;
use std::io::Read;
use thiserror::Error;

#[derive(Error, Debug)]
pub enum ChunkReadError {
    #[error("chunk not generated at {0:?}")]
    NotGenerated(ChunkPos),
    #[error("unknown compression type {0}")]
    UnknownCompression(u8),
    #[error("decompression failed: {0}")]
    Decompression(#[from] std::io::Error),
    #[error("NBT parse failed: {0}")]
    NbtParse(#[from] fastnbt::error::Error),
}

pub fn read_chunk(region: &RegionFile, local_x: u8, local_z: u8, pos: ChunkPos)
    -> Result<Value, ChunkReadError>
{
    let (compression, data) = region
        .raw_chunk_data(local_x, local_z)
        .ok_or(ChunkReadError::NotGenerated(pos))?;

    let nbt_bytes = decompress(compression, data)?;
    let value = fastnbt::from_bytes(&nbt_bytes)?;
    Ok(value)
}

fn decompress(compression_type: u8, data: &[u8]) -> Result<Vec<u8>, std::io::Error> {
    match compression_type {
        1 => {  // GZip (legacy, rare)
            let mut buf = Vec::with_capacity(data.len() * 4);
            GzDecoder::new(data).read_to_end(&mut buf)?;
            Ok(buf)
        }
        2 => {  // Zlib (standard)
            let mut buf = Vec::with_capacity(data.len() * 4);
            ZlibDecoder::new(data).read_to_end(&mut buf)?;
            Ok(buf)
        }
        3 => Ok(data.to_vec()),  // Uncompressed
        n => Err(std::io::Error::new(
            std::io::ErrorKind::InvalidData,
            format!("unknown compression type {n}"),
        )),
    }
}
```

---

## Rule: `mc-nbt-typed` — Use typed NBT accessors, not `Value::get()`

`fastnbt::Value` is an untyped enum. Use `fastnbt::de` with serde for production code:

```rust
use fastnbt::error::Result as NbtResult;
use serde::Deserialize;

/// Typed representation of a chunk's NBT data root
#[derive(Deserialize, Debug)]
pub struct ChunkNbt {
    #[serde(rename = "DataVersion")]
    pub data_version: i32,
    #[serde(rename = "xPos")]
    pub x_pos: i32,
    #[serde(rename = "zPos")]
    pub z_pos: i32,
    #[serde(rename = "yPos")]
    pub y_pos: i32,   // section Y offset (−4 for 1.18+)
    #[serde(rename = "Status")]
    pub status: String,
    #[serde(rename = "sections")]
    pub sections: Vec<SectionNbt>,
}

#[derive(Deserialize, Debug)]
pub struct SectionNbt {
    #[serde(rename = "Y")]
    pub y: i8,
    pub block_states: Option<BlockStatesNbt>,
    pub biomes: Option<BiomesNbt>,
}

#[derive(Deserialize, Debug)]
pub struct BlockStatesNbt {
    pub palette: Vec<BlockPaletteEntry>,
    pub data: Option<fastnbt::LongArray>,  // packed bit array
}

#[derive(Deserialize, Debug)]
pub struct BlockPaletteEntry {
    #[serde(rename = "Name")]
    pub name: String,         // "minecraft:stone"
    #[serde(rename = "Properties")]
    pub properties: Option<AHashMap<String, String>>,
}

// Usage
pub fn parse_chunk(nbt_bytes: &[u8]) -> NbtResult<ChunkNbt> {
    fastnbt::from_bytes(nbt_bytes)
}
```

---

## Rule: `mc-nbt-snbt-display` — SNBT for human-readable NBT output

SNBT (Stringified NBT) is used in commands like `/data get`. Useful for debugging:

```rust
use fastnbt::Value;

/// Convert NBT Value to SNBT string (for debugging / commands)
pub fn to_snbt(value: &Value) -> String {
    match value {
        Value::Byte(n)    => format!("{n}b"),
        Value::Short(n)   => format!("{n}s"),
        Value::Int(n)     => format!("{n}"),
        Value::Long(n)    => format!("{n}L"),
        Value::Float(f)   => format!("{f}f"),
        Value::Double(d)  => format!("{d}d"),
        Value::String(s)  => format!("\"{}\"", s.replace('"', "\\\"")),
        Value::List(items) => {
            let inner: Vec<String> = items.iter().map(to_snbt).collect();
            format!("[{}]", inner.join(","))
        }
        Value::Compound(map) => {
            let inner: Vec<String> = map.iter()
                .map(|(k, v)| format!("{k}:{}", to_snbt(v)))
                .collect();
            format!("{{{}}}", inner.join(","))
        }
        Value::ByteArray(b) => format!("[B;{}]", b.iter().map(|n| format!("{n}b")).collect::<Vec<_>>().join(",")),
        Value::IntArray(i)  => format!("[I;{}]", i.iter().map(|n| n.to_string()).collect::<Vec<_>>().join(",")),
        Value::LongArray(l) => format!("[L;{}]", l.iter().map(|n| format!("{n}L")).collect::<Vec<_>>().join(",")),
    }
}
```

---

## Error Handling Pattern for World I/O

```rust
// ❌ Wrong: panic on malformed world data
fn load_chunk(region: &RegionFile, pos: ChunkPos) -> Chunk {
    let data = region.raw_chunk_data(pos.x as u8, pos.z as u8).unwrap();
    let nbt: Value = fastnbt::from_bytes(&data.1).unwrap();  // crashes on corruption
    // ...
}

// ✅ Correct: propagate errors, let caller decide recovery strategy
fn load_chunk(region: &RegionFile, pos: ChunkPos)
    -> Result<Chunk, ChunkReadError>
{
    let local_x = (pos.x.rem_euclid(32)) as u8;
    let local_z = (pos.z.rem_euclid(32)) as u8;
    let (compression, data) = region
        .raw_chunk_data(local_x, local_z)
        .ok_or(ChunkReadError::NotGenerated(pos))?;
    let nbt_bytes = decompress(compression, data)?;
    let nbt: ChunkNbt = fastnbt::from_bytes(&nbt_bytes)?;
    Ok(Chunk::from_nbt(nbt))
}

// Caller: log and regenerate corrupted chunks instead of crashing
async fn ensure_chunk(world: &World, pos: ChunkPos) -> Arc<RwLock<Chunk>> {
    match world.load_chunk_from_disk(pos).await {
        Ok(chunk) => chunk,
        Err(ChunkReadError::NotGenerated(_)) => world.generate_chunk(pos).await,
        Err(e) => {
            tracing::warn!("Corrupted chunk {pos:?}: {e} — regenerating");
            world.generate_chunk(pos).await  // graceful recovery
        }
    }
}
```

> **leonardomso rules**: `err-result-over-panic` — return Result, never panic on expected errors.
> `err-thiserror-lib` — use thiserror for error type definition.
> `err-context-chain` — add context to errors for better debugging.
