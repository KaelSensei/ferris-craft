# ferris-craft — Installation Guide

> Rust skill set for Minecraft-domain systems.  
> Follow the steps below to set up the repository from the downloaded files.

**This repo is already set up:** the structure in §2 lives at the repo root (`SKILL.md`, `CLAUDE.md`, `AGENTS.md`, `README.md`, and `skills/` with all skill files).

---

## 1. Create the repo structure

Create a folder called `ferris-craft` and inside it a `skills/` subfolder:

```
ferris-craft/
└── skills/
```

---

## 2. Place the files

| File | Where it goes |
|---|---|
| `SKILL.md` | `ferris-craft/SKILL.md` |
| `CLAUDE.md` | `ferris-craft/CLAUDE.md` |
| `AGENTS.md` | `ferris-craft/AGENTS.md` |
| `README.md` | `ferris-craft/README.md` |
| `domain-minecraft.md` | `ferris-craft/skills/domain-minecraft.md` |
| `mc-00-vibe.md` | `ferris-craft/skills/mc-00-vibe.md` |
| `mc-01-ownership.md` | `ferris-craft/skills/mc-01-ownership.md` |
| `mc-02-chunk-storage.md` | `ferris-craft/skills/mc-02-chunk-storage.md` |
| `mc-03-worldgen.md` | `ferris-craft/skills/mc-03-worldgen.md` |
| `mc-04-entity-ecs.md` | `ferris-craft/skills/mc-04-entity-ecs.md` |
| `mc-05-nbt-io.md` | `ferris-craft/skills/mc-05-nbt-io.md` |
| `mc-06-networking.md` | `ferris-craft/skills/mc-06-networking.md` |
| `mc-07-performance.md` | `ferris-craft/skills/mc-07-performance.md` |
| `mc-08-lighting.md` | `ferris-craft/skills/mc-08-lighting.md` |
| `mc-10-plugins.md` | `ferris-craft/skills/mc-10-plugins.md` |

Your final structure should look like this:

```
ferris-craft/
├── SKILL.md
├── CLAUDE.md
├── AGENTS.md
├── README.md
└── skills/
    ├── domain-minecraft.md
    ├── mc-00-vibe.md
    ├── mc-01-ownership.md
    ├── mc-02-chunk-storage.md
    ├── mc-03-worldgen.md
    ├── mc-04-entity-ecs.md
    ├── mc-05-nbt-io.md
    ├── mc-06-networking.md
    ├── mc-07-performance.md
    ├── mc-08-lighting.md
    └── mc-10-plugins.md
```

> The `ferris-craft-cursor-setup...` file in your `files/` folder can be deleted — it is not needed.

---

## 3. Push to GitHub

```bash
cd ferris-craft
git init
git add .
git commit -m "feat: initial ferris-craft skill set"
gh repo create ferris-craft --public --source=. --push
```

> `gh` is the GitHub CLI — install it from [cli.github.com](https://cli.github.com) if you don't have it yet.

---

## 4. Install in your AI agent

**Claude Code — global (applies to all your projects):**
```bash
git clone https://github.com/YOU/ferris-craft ~/.claude/skills/ferris-craft
```

**Claude Code — project only:**
```bash
git clone https://github.com/YOU/ferris-craft .claude/skills/ferris-craft
```

**Cursor:**
```bash
cp AGENTS.md .cursorrules
```

**GitHub Copilot:**
```bash
mkdir -p .github
cp AGENTS.md .github/copilot-instructions.md
```

**Any other agent (Codex, Windsurf, Aider…):**
```bash
cp AGENTS.md AGENTS.md  # already at repo root — most agents pick it up automatically
```
