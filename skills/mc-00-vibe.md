---
name: mc-00-vibe
description: >
  Entry skill for non-technical users and vibe coders. Use this when the person
  describes what they WANT in plain English rather than technical terms.
  Translates their goal into the right technical skill(s) automatically.
  Triggers on anything Minecraft + Rust flavored but non-technical:
  "I want to make a Minecraft server", "how do I make a world generator",
  "my server is laggy", "I want players to be able to move", etc.
layer: 0
triggers:
  - I want to make, how do I make, my server, I want players, laggy, slow server
  - beginner, I don't know Rust, help me start, vibe, where do I start
---

# mc-00 — Start Here (Vibe Coder Edition)

> You don't need to know Rust deeply to build Minecraft-like things with it.
> Just describe what you want — this skill figures out the rest.

---

## What do you want to build?

Tell the AI in plain English. Here are the most common goals and what they map to behind the scenes:

---

### 🌍 "I want to generate a world / terrain"

**What you mean:** You want procedurally generated land, mountains, oceans, caves — like Minecraft's world creation.

**What happens under the hood:** noise functions (Perlin / simplex) produce a height value for every X/Z coordinate. Positive density = solid block, negative = air.

**Skill invoked:** → `mc-03-worldgen`

**Just ask:** *"Generate a basic terrain with hills and flat areas for a 16×16 chunk"*

---

### 🧱 "I want to store / read / change blocks in the world"

**What you mean:** You want a world made of blocks you can place, break, and query — like `getBlock(x, y, z)`.

**What happens under the hood:** blocks are stored in 16×16×16 sections with a compressed palette. Think of it like a very efficient 3D array.

**Skill invoked:** → `mc-02-chunk-storage`

**Just ask:** *"Give me a chunk data structure where I can get and set blocks by position"*

---

### 🧍 "I want players / mobs to move around"

**What you mean:** You want entities that walk, run, jump, fall, and collide with blocks — basic physics.

**What happens under the hood:** each entity has a position and velocity. Every tick (50ms), gravity pulls them down, drag slows them, and collision detection stops them going through blocks.

**Skill invoked:** → `mc-04-entity-ecs`

**Just ask:** *"Give me a player physics system with gravity, jumping, and basic block collision"*

---

### 🖧 "I want to make a multiplayer server"

**What you mean:** You want multiple players to connect, see each other, and interact in the same world.

**What happens under the hood:** a TCP server accepts connections, reads packets from players, and feeds them into a game loop that runs 20 times per second.

**Skill invoked:** → `mc-06-networking`

**Just ask:** *"Give me the basic skeleton of a Minecraft-compatible server that accepts player connections"*

---

### 📦 "I want to load / save the world from disk"

**What you mean:** You want the world to persist — save it when the server stops, load it back when it starts.

**What happens under the hood:** Minecraft uses a binary format called NBT, compressed with zlib, stored in `.mca` region files. Each file holds a 32×32 grid of chunks.

**Skill invoked:** → `mc-05-nbt-io`

**Just ask:** *"Give me code to read a chunk from a .mca region file"*

---

### ⚡ "My server is slow / laggy"

**What you mean:** Things feel slow, players notice delays, or it crashes under load.

**What happens under the hood:** the game loop has a strict 50ms budget. If any single system (entities, world gen, block updates) takes too long, the whole server falls behind.

**Skill invoked:** → `mc-07-performance`

**Just ask:** *"My entity tick is taking too long — how do I profile it and fix the most common causes?"*

---

### 🔴 "I want redstone-like logic / circuits"

**What you mean:** You want blocks that can signal each other — like switches, doors, and logic gates.

**What happens under the hood:** redstone uses a wave-based update system. Only blocks that *changed* trigger their neighbours — not the whole world every tick.

**Skill invoked:** → `mc-07-performance` (BFS redstone section)

**Just ask:** *"Give me a simple redstone-style signal propagation system"*

---

### 🗺️ "I want biomes / different terrain types"

**What you mean:** You want different regions — desert, forest, ocean — blending into each other.

**What happens under the hood:** two extra noise layers (continentalness and weirdness) determine which biome template applies at each XZ position.

**Skill invoked:** → `mc-03-worldgen`

**Just ask:** *"Add biome variation to my terrain generator — at least 3 different biome types"*

---

## Starter project — copy this to begin

If you have zero code yet, paste this into your AI chat to get a working skeleton:

```
I'm building a Minecraft-like game in Rust. I'm not an expert Rust developer.
Please give me a minimal working project with:
- A chunk that can store blocks (16x16x16)
- A function to set and get blocks by x/y/z
- A simple flat world generator that fills the bottom half with stone and top with air
- A main.rs that creates a world, generates one chunk, and prints the block at 0,0,0

Keep the code simple and well commented. Use the ferris-craft skill set.
```

---

## Things that will confuse you at first (and why they're fine)

| Rust thing | What it means in plain English |
|---|---|
| `Arc<RwLock<Chunk>>` | "This chunk can be safely shared between multiple parts of the code at the same time" |
| `async / await` | "Do this in the background without freezing the rest of the server" |
| `Result<T, E>` | "This might fail — here's the success value OR the error" |
| `#[derive(Component)]` | "This data belongs to an entity in the ECS system" |
| `par_iter()` | "Process all of these at the same time using multiple CPU cores" |
| `u16` for block states | "A block state ID — just a number between 0 and 25,000" |

---

## The one rule that will save you most headaches

> **Never put blocking code (file I/O, network calls) inside the game tick loop.**
>
> The game loop runs 20 times per second and has 50ms to do everything.
> Loading a file takes hundreds of milliseconds. Always do it in the background.

If you see your server freezing when players move around: this is almost certainly the cause.
