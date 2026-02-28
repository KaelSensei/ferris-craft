# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## [Unreleased]

### Added

- Initial ferris-craft (Rust Skills — Minecraft Domain) skill set: root docs (`SKILL.md`, `CLAUDE.md`, `AGENTS.md`, `README.md`) and `skills/` with 8 skill files (domain-minecraft, mc-01 through mc-07) for voxel/Minecraft-adjacent Rust systems.
- Main README banner image: `assets/rust-skills-minecraft-banner.png` (Rust Skills IA for Minecraft — in-game Rust companion), displayed at top of README.
- **mc-00-vibe** (Layer 0): New entry skill `skills/mc-00-vibe.md` for non-technical / vibe coders — translates "I want to make…", "my server is laggy", "where do I start" into the right technical skill. Includes plain-English goal table and starter project snippet.
- **mc-08-lighting** (Layer 2): New skill `skills/mc-08-lighting.md` — Minecraft-accurate lighting (heightmap skylight init, BFS flood-fill for sky/block light, removal pass, cross-chunk borders). References Starlight/Phosphor and wiki spec.
- **mc-10-plugins** (Layer 2): New skill `skills/mc-10-plugins.md` — Plugin architecture: trait objects (same binary), libloading (dynamic .so), WASM via wasmtime (sandboxed). Event bus pattern and comparison table.
- **domain-minecraft**: Ecosystem map updated (ferrumc, feather abandoned, bedrock-rs, lighting and plugin sections); routing extended for mc-08 and mc-10.
