---
name: mc-10-plugins
description: >
  Layer 2 — Design Choices (WHAT). Plugin and extension architecture for Minecraft
  servers in Rust. Covers trait-object plugins (safe, same binary), dynamic loading
  via libloading (unsafe, separate .so), and WASM sandboxing via wasmtime (safe,
  cross-platform, isolated). Use when you want server behaviour to be extensible
  without recompiling the core.
layer: 2
triggers:
  - plugin, extension, mod, scripting, dynamic loading, WASM, wasmtime, libloading
  - extensible server, plugin system, event bus, hook, scripting API
---

# mc-10 — Plugin & Extension Architecture

> **Core Question**: *How do you let others extend the server without giving them
> access to unsafe internals, without recompiling the whole binary, and without
> crashing the tick thread when their code panics?*
>
> Rust has no stable ABI and no runtime reflection — you must choose your
> isolation boundary deliberately.

---

## Architecture Decision Tree

```
Do plugins need to run in a separate process / be sandboxed?
├── YES → WASM via wasmtime (safe, cross-platform, no ABI issues)
│         Best for: untrusted community plugins, hot-reload, Bedrock parity
│
└── NO  → Do plugins live in a separate .so/.dll file?
          ├── YES → libloading + explicit C ABI (unsafe, same platform only)
          │         Best for: performance-critical internal plugins you control
          │
          └── NO  → Trait objects in same binary (fully safe, simplest)
                    Best for: built-in gameplay modes, feature flags, prototyping
```

---

## Approach 1 — Trait Object Plugins (Same Binary)

The simplest approach. All plugins are compiled into the server binary.
Zero unsafe code. Recommended for most projects.

```rust
use std::sync::Arc;

// ── EVENT TYPES ──────────────────────────────────────────────────────────────

/// Events the server emits to plugins.
#[non_exhaustive]  // allows adding variants without breaking plugins
pub enum ServerEvent<'a> {
    PlayerJoin   { player: &'a Player },
    PlayerLeave  { player: &'a Player, reason: &'a str },
    BlockPlace   { player: &'a Player, pos: BlockPos, block: BlockState },
    BlockBreak   { player: &'a Player, pos: BlockPos, block: BlockState },
    ChatMessage  { player: &'a Player, message: &'a str },
    EntityDamage { entity: EntityId, damage: f32, source: DamageSource },
    ServerTick   { tick: u64 },
}

/// A plugin's response to an event.
pub enum EventResult {
    /// Let the event proceed normally
    Pass,
    /// Cancel the event (e.g. prevent block placement)
    Cancel,
    /// Cancel and send a message to the player
    CancelWith(String),
}

// ── PLUGIN TRAIT ─────────────────────────────────────────────────────────────

/// The core plugin interface — implement this to create a plugin.
pub trait Plugin: Send + Sync {
    /// Unique identifier, e.g. "essentials" or "anti-grief"
    fn name(&self) -> &str;

    /// Called once when the server starts, before the first tick
    fn on_enable(&mut self, api: &PluginApi) {}

    /// Called once when the server shuts down gracefully
    fn on_disable(&mut self) {}

    /// Called for every server event — return Pass to not interfere
    fn on_event(&mut self, event: &ServerEvent<'_>) -> EventResult {
        EventResult::Pass
    }
}

// ── PLUGIN REGISTRY ──────────────────────────────────────────────────────────

pub struct PluginRegistry {
    plugins: Vec<Box<dyn Plugin>>,
}

impl PluginRegistry {
    pub fn new() -> Self { Self { plugins: Vec::new() } }

    pub fn register(&mut self, plugin: impl Plugin + 'static) {
        self.plugins.push(Box::new(plugin));
    }

    /// Dispatch an event to all plugins; stop if any cancels it.
    pub fn dispatch(&mut self, event: &ServerEvent<'_>) -> EventResult {
        for plugin in &mut self.plugins {
            match plugin.on_event(event) {
                EventResult::Pass => {}
                result => return result,  // first cancellation wins
            }
        }
        EventResult::Pass
    }
}

// ── PLUGIN API (what plugins can call back into) ──────────────────────────────

/// Safe API surface exposed to plugins — no direct World access.
/// Acts as a capability boundary: plugins only do what the API allows.
pub struct PluginApi {
    pub world: Arc<WorldHandle>,       // limited view of the world
    pub scheduler: TaskScheduler,      // schedule future tasks
    pub commands: CommandRegistry,     // register /commands
}

// ── EXAMPLE PLUGIN ───────────────────────────────────────────────────────────

struct AntiGriefPlugin {
    protected_regions: Vec<Aabb>,
}

impl Plugin for AntiGriefPlugin {
    fn name(&self) -> &str { "anti-grief" }

    fn on_event(&mut self, event: &ServerEvent<'_>) -> EventResult {
        if let ServerEvent::BlockBreak { player, pos, .. } = event {
            for region in &self.protected_regions {
                if region.contains(*pos) {
                    return EventResult::CancelWith(
                        "This area is protected.".into()
                    );
                }
            }
        }
        EventResult::Pass
    }
}

// Registration in main:
// registry.register(AntiGriefPlugin { protected_regions: vec![spawn_region] });
```

---

## Approach 2 — Dynamic Loading via `libloading` (Separate .so/.dll)

Use when you want plugins as separate files that can be distributed independently.
**Requires explicit C ABI (`extern "C"`) — Rust has no stable ABI.**

```toml
# Plugin Cargo.toml
[lib]
crate-type = ["cdylib"]  # produces .so / .dll

[dependencies]
ferris_craft_api = { path = "../ferris-craft-api" }  # shared API crate
```

```rust
// ── PLUGIN SIDE (.so) ────────────────────────────────────────────────────────

use ferris_craft_api::{Plugin, PluginVTable};

struct MyPlugin;

impl Plugin for MyPlugin {
    fn name(&self) -> &str { "my-plugin" }
    fn on_event(&mut self, event: &ServerEvent<'_>) -> EventResult { EventResult::Pass }
}

/// This is the ONLY symbol the server looks for in the .so file.
/// extern "C" + no_mangle = stable, findable by libloading.
#[no_mangle]
pub extern "C" fn create_plugin() -> *mut dyn Plugin {
    Box::into_raw(Box::new(MyPlugin))
}

// ── SERVER SIDE (loads the .so) ──────────────────────────────────────────────

use libloading::{Library, Symbol};

pub struct DynamicPlugin {
    _lib: Library,           // must outlive the plugin pointer
    plugin: Box<dyn Plugin>,
}

impl DynamicPlugin {
    /// Load a plugin from a .so / .dll path.
    ///
    /// # Safety
    /// The .so must export `create_plugin` with the correct signature.
    /// The plugin must be compiled with the SAME version of ferris_craft_api.
    pub unsafe fn load(path: &Path) -> anyhow::Result<Self> {
        let lib = Library::new(path)?;

        // Look up the factory function by name
        let constructor: Symbol<unsafe extern "C" fn() -> *mut dyn Plugin> =
            lib.get(b"create_plugin")?;

        let raw = constructor();
        if raw.is_null() {
            anyhow::bail!("create_plugin returned null");
        }

        Ok(Self {
            plugin: Box::from_raw(raw),
            _lib: lib,  // keep Library alive — dropping it would unload the .so
        })
    }
}

// ── VERSION GUARD ─────────────────────────────────────────────────────────────
// Prevent ABI mismatch crashes by embedding an API version in every plugin.

pub const API_VERSION: u32 = 1;

#[no_mangle]
pub extern "C" fn api_version() -> u32 { ferris_craft_api::API_VERSION }

// Server checks this BEFORE calling create_plugin:
// if api_version() != API_VERSION { reject plugin }
```

> ⚠️ **libloading caveats**
> - Plugins MUST be compiled with the exact same Rust compiler version and API crate
> - Panics inside a plugin unwind across the FFI boundary → **undefined behaviour**
> - Wrap every plugin call in `std::panic::catch_unwind` on the server side
> - Hot-reload (unload + reload) is theoretically possible but risky — prefer WASM

---

## Approach 3 — WASM Sandbox via `wasmtime` (Recommended for Untrusted Plugins)

Plugins compile to WASM. The server runs them in a sandbox.
Panics, infinite loops, and bad memory accesses are all caught safely.

```toml
# Server Cargo.toml
[dependencies]
wasmtime = "18"
wasmtime-wasi = "18"
```

```rust
use wasmtime::*;
use wasmtime_wasi::WasiCtxBuilder;

pub struct WasmPlugin {
    store: Store<WasiCtx>,
    instance: Instance,
    /// Pre-compiled function exports
    on_event: TypedFunc<(i32,), i32>,
    on_enable: TypedFunc<(), ()>,
}

impl WasmPlugin {
    pub fn load(engine: &Engine, wasm_bytes: &[u8]) -> anyhow::Result<Self> {
        let module = Module::new(engine, wasm_bytes)?;

        // WASI context: controls what the plugin can access
        // Here: NO filesystem, NO network — fully sandboxed
        let wasi = WasiCtxBuilder::new()
            .inherit_stdio()   // allow println! for debugging
            .build();

        let mut store = Store::new(engine, wasi);
        let mut linker = Linker::new(engine);
        wasmtime_wasi::add_to_linker(&mut linker, |s| s)?;

        // Expose server API functions to the plugin
        linker.func_wrap("env", "send_message", |mut caller: Caller<'_, _>, player_id: i32, msg_ptr: i32, msg_len: i32| {
            // Read message string from plugin's WASM memory
            let memory = caller.get_export("memory")
                .and_then(|e| e.into_memory())
                .unwrap();
            let data = memory.data(&caller);
            let msg = std::str::from_utf8(
                &data[msg_ptr as usize..(msg_ptr + msg_len) as usize]
            ).unwrap_or("");
            // Send message to player — call back into server
            println!("Plugin → player {player_id}: {msg}");
        })?;

        let instance = linker.instantiate(&mut store, &module)?;

        let on_event  = instance.get_typed_func::<(i32,), i32>(&mut store, "on_event")?;
        let on_enable = instance.get_typed_func::<(), ()>(&mut store, "on_enable")?;

        Ok(Self { store, instance, on_event, on_enable })
    }

    /// Call the plugin's on_event. Panics in plugin code are caught by wasmtime.
    pub fn dispatch_event(&mut self, event_id: i32) -> EventResult {
        match self.on_event.call(&mut self.store, (event_id,)) {
            Ok(0) => EventResult::Pass,
            Ok(_) => EventResult::Cancel,
            Err(e) => {
                // Plugin trapped (panic, OOM, etc.) — server is unaffected
                tracing::warn!("WASM plugin error: {e}");
                EventResult::Pass
            }
        }
    }
}
```

```toml
# Plugin Cargo.toml (the plugin author writes this)
[lib]
crate-type = ["cdylib"]

[dependencies]
# No std — WASM plugins are typically no_std or use wasm-specific std

[[target.'cfg(target_arch = "wasm32")'.dependencies]]
wasm-bindgen = "0.2"
```

```rust
// Plugin source (compiled to .wasm by the author)
#[no_mangle]
pub extern "C" fn on_event(event_id: i32) -> i32 {
    match event_id {
        EVENT_PLAYER_JOIN => {
            send_message(0, "Welcome to the server!");
            0  // Pass
        }
        _ => 0,
    }
}

extern "C" {
    fn send_message(player_id: i32, msg: *const u8, msg_len: i32);
}
```

---

## Comparison Table

| | Trait Objects | libloading | WASM (wasmtime) |
|---|---|---|---|
| Safety | ✅ Fully safe | ⚠️ Unsafe FFI | ✅ Sandboxed |
| Plugin panic isolation | ❌ Crashes server | ❌ UB across FFI | ✅ Caught by runtime |
| Hot reload | ❌ No | ⚠️ Risky | ✅ Yes |
| Cross-platform plugins | ✅ (same binary) | ❌ Platform-specific | ✅ WASM is universal |
| Performance overhead | 0 | ~0 | ~10–20% vs native |
| Plugin can be written in | Rust only | Rust only | Rust, C, Go, JS… |
| ABI stability | N/A | ❌ Fragile | ✅ Stable WASM ABI |
| Complexity | Low | Medium | High |
| **Recommended for** | Built-in features | Internal perf plugins | Community plugins |

---

## Event Bus Pattern (works with all 3 approaches)

```rust
/// Typed event bus — avoids giant match arms in every plugin.
pub struct EventBus {
    handlers: AHashMap<TypeId, Vec<Box<dyn Any + Send + Sync>>>,
}

impl EventBus {
    pub fn subscribe<E: 'static>(&mut self, handler: impl Fn(&E) + Send + Sync + 'static) {
        self.handlers
            .entry(TypeId::of::<E>())
            .or_default()
            .push(Box::new(handler));
    }

    pub fn publish<E: 'static>(&self, event: &E) {
        if let Some(handlers) = self.handlers.get(&TypeId::of::<E>()) {
            for handler in handlers {
                if let Some(f) = handler.downcast_ref::<fn(&E)>() {
                    f(event);
                }
            }
        }
    }
}

// Usage:
bus.subscribe::<PlayerJoinEvent>(|e| println!("{} joined", e.player.name));
bus.publish(&PlayerJoinEvent { player: &player });
```

---

## Cargo.toml for Plugin System

```toml
[dependencies]
# Dynamic loading
libloading = "0.8"

# WASM sandbox
wasmtime      = { version = "18", optional = true }
wasmtime-wasi = { version = "18", optional = true }

[features]
wasm-plugins = ["wasmtime", "wasmtime-wasi"]
```
