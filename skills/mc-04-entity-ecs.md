---
name: mc-04-entity-ecs
description: >
  Layer 2 — Design Choices (WHAT). Entity simulation via ECS: player movement,
  mob AI, physics (AABB collision, gravity, drag), pathfinding (A* with chunk-aware
  heuristics), and the tick loop. Use when building entity systems, player movement,
  mob behavior, or anything that touches the entity simulation loop.
layer: 2
triggers:
  - entity, player, mob, movement, physics, AABB, collision, pathfinding, AI
  - tick loop, ECS, bevy, entity system, gravity, velocity, jump
---

# mc-04 — Entity Simulation & ECS

> **Core Question**: *Why is ECS non-negotiable for Minecraft entity simulation?*
>
> Minecraft has ~100 entity types sharing ~40 behavioral components (can-move, has-health,
> can-swim, on-fire, …). OOP inheritance creates diamond inheritance nightmares.
> **ECS = composition + cache-local component arrays + auto-parallelism.**

---

## ECS Component Design

```rust
use bevy::prelude::*;

// ── POSITION & PHYSICS ──────────────────────────────────────────────────────

/// World position — f64 to avoid "far lands" precision loss beyond ±2048 blocks
#[derive(Component, Debug, Clone, Copy)]
pub struct Position(pub DVec3);

/// Current velocity in blocks/tick
#[derive(Component, Debug, Clone, Copy, Default)]
pub struct Velocity(pub DVec3);

/// Whether the entity is currently touching the ground
#[derive(Component, Debug, Clone, Copy, Default)]
pub struct OnGround(pub bool);

/// AABB half-extents: width/2, height, width/2 (Minecraft convention)
#[derive(Component, Debug, Clone, Copy)]
pub struct BoundingBox {
    pub half_width: f64,   // typical player: 0.3
    pub height: f64,       // typical player: 1.8
}

// ── SIMULATION STATE ─────────────────────────────────────────────────────────

/// How many ticks this entity has been alive
#[derive(Component, Default)]
pub struct TickAge(pub u64);

/// Entity health (0.0 = dead)
#[derive(Component)]
pub struct Health { pub current: f32, pub max: f32 }

/// Flags for special physics states
#[derive(Component, Default)]
pub struct PhysicsFlags {
    pub in_water:  bool,
    pub in_lava:   bool,
    pub on_fire:   bool,
    pub sprinting: bool,
    pub sneaking:  bool,
    pub flying:    bool,  // creative mode
}

// ── PLAYER-SPECIFIC ──────────────────────────────────────────────────────────

/// Marks an entity as a player (used as filter in queries)
#[derive(Component)]
pub struct Player {
    pub username: Box<str>,  // Box<str> not String — immutable after login, no realloc
    pub uuid: u128,
}

/// Player input state — populated from incoming packets
#[derive(Component, Default)]
pub struct PlayerInput {
    pub forward: bool, pub backward: bool,
    pub left: bool, pub right: bool,
    pub jump: bool, pub sneak: bool,
    pub sprint: bool,
    pub yaw: f32, pub pitch: f32,   // looking direction
}

// ── MOB AI ───────────────────────────────────────────────────────────────────

/// Current AI goal for a mob
#[derive(Component)]
pub enum MobGoal {
    Idle,
    Wander { target: DVec3, recalculate_in: u32 },
    Follow { target: EntityId, max_distance: f64 },
    Attack { target: EntityId, cooldown: u32 },
    Flee   { from: EntityId, until_distance: f64 },
}
```

---

## Rule: `mc-ecs-physics-system` — Physics tick system

```rust
/// Minecraft-accurate physics for all entities with velocity.
///
/// Runs every tick (50ms) via Bevy's scheduler.
/// Bevy auto-parallelises when no component aliasing exists.
pub fn physics_tick(
    mut query: Query<(
        &mut Position,
        &mut Velocity,
        &mut OnGround,
        &PhysicsFlags,
        &BoundingBox,
    )>,
    world_data: Res<WorldData>,
) {
    // par_iter_mut: Bevy uses rayon to process all entities in parallel
    query.par_iter_mut().for_each(|(mut pos, mut vel, mut on_ground, flags, bbox)| {
        // 1. Apply gravity (skip if flying)
        if !flags.flying {
            vel.0.y -= 0.08;        // Minecraft gravity constant (blocks/tick²)
            vel.0.y *= 0.98;        // Vertical drag
        }

        // 2. Apply horizontal drag
        let drag = if on_ground.0 {
            if flags.sprinting { 0.546 } else { 0.6 }
        } else if flags.in_water { 0.8 } else { 0.91 };
        vel.0.x *= drag;
        vel.0.z *= drag;

        // 3. Resolve AABB collisions against world geometry
        let (resolved_vel, ground) = resolve_aabb_collision(
            pos.0, vel.0, bbox, &world_data
        );

        // 4. Apply movement
        pos.0 += resolved_vel;
        vel.0 = resolved_vel;
        on_ground.0 = ground;

        // 5. Clamp velocity to max speed (prevents tunneling through blocks)
        const MAX_VEL: f64 = 100.0;
        vel.0 = vel.0.clamp(DVec3::splat(-MAX_VEL), DVec3::splat(MAX_VEL));
    });
}

/// AABB sweep-based collision resolution.
/// Returns (resolved velocity, is_on_ground).
fn resolve_aabb_collision(
    pos: DVec3,
    vel: DVec3,
    bbox: &BoundingBox,
    world: &WorldData,
) -> (DVec3, bool) {
    let aabb = Aabb::from_center_bbox(pos, bbox);
    let expanded = aabb.expand(vel); // Minkowski sum for sweep test

    // Collect all blocks the expanded AABB overlaps
    let candidate_blocks = world.blocks_in_aabb(&expanded);

    // Sweep Y first (gravity), then X, then Z — matches Minecraft order
    let (vel_y, on_ground) = sweep_y(aabb, vel.y, &candidate_blocks, world);
    let vel_x = sweep_x(aabb.translate(DVec3::new(0.0, vel_y, 0.0)), vel.x, &candidate_blocks, world);
    let vel_z = sweep_z(aabb.translate(DVec3::new(vel_x, vel_y, 0.0)), vel.z, &candidate_blocks, world);

    (DVec3::new(vel_x, vel_y, vel_z), on_ground)
}
```

---

## Rule: `mc-ecs-player-input` — Player input system (packet-driven)

```rust
/// Process incoming player movement packets and update PlayerInput component.
/// Runs before physics_tick in the schedule.
pub fn player_input_system(
    mut query: Query<(&mut PlayerInput, &Player)>,
    mut packet_events: EventReader<PlayerMovePacket>,
) {
    for packet in packet_events.read() {
        // Find the player entity this packet belongs to
        for (mut input, player) in query.iter_mut() {
            if player.uuid == packet.player_uuid {
                input.forward   = packet.flags.contains(MoveFlags::FORWARD);
                input.backward  = packet.flags.contains(MoveFlags::BACKWARD);
                input.left      = packet.flags.contains(MoveFlags::STRAFE_LEFT);
                input.right     = packet.flags.contains(MoveFlags::STRAFE_RIGHT);
                input.jump      = packet.flags.contains(MoveFlags::JUMP);
                input.sneak     = packet.flags.contains(MoveFlags::SNEAK);
                input.sprint    = packet.flags.contains(MoveFlags::SPRINT);
                input.yaw       = packet.yaw;
                input.pitch     = packet.pitch;
            }
        }
    }
}

/// Convert player input into velocity adjustments.
/// Runs after player_input_system, before physics_tick.
pub fn apply_player_input(
    mut query: Query<(&PlayerInput, &mut Velocity, &OnGround, &PhysicsFlags), With<Player>>,
) {
    for (input, mut vel, on_ground, flags) in query.iter_mut() {
        let speed = if flags.sprinting { 0.13 } else { 0.1 };
        let yaw_rad = input.yaw.to_radians();

        // Strafe direction in world space
        let forward_x = -yaw_rad.sin() as f64;
        let forward_z = -yaw_rad.cos() as f64;
        let strafe_x  =  yaw_rad.cos() as f64;
        let strafe_z  = -yaw_rad.sin() as f64;

        let mut move_x = 0.0_f64;
        let mut move_z = 0.0_f64;
        if input.forward  { move_x += forward_x; move_z += forward_z; }
        if input.backward { move_x -= forward_x; move_z -= forward_z; }
        if input.left     { move_x -= strafe_x;  move_z -= strafe_z; }
        if input.right    { move_x += strafe_x;  move_z += strafe_z; }

        // Normalize diagonal movement (no 1.41× speed boost)
        let len = (move_x * move_x + move_z * move_z).sqrt();
        if len > 0.0 {
            vel.0.x += (move_x / len) * speed;
            vel.0.z += (move_z / len) * speed;
        }

        // Jump
        if input.jump && on_ground.0 {
            vel.0.y = 0.42;  // Minecraft jump velocity constant
        }
    }
}
```

---

## Rule: `mc-ecs-pathfinding` — A* pathfinding with chunk-aware heuristics

```rust
use std::collections::BinaryHeap;
use std::cmp::Reverse;

pub struct PathFinder;

impl PathFinder {
    /// A* pathfinding for mob navigation.
    /// Max nodes: 200 (matches Minecraft's mob path budget).
    pub fn find_path(
        start: BlockPos,
        goal: BlockPos,
        world: &WorldData,
        max_nodes: usize,
    ) -> Option<Vec<BlockPos>> {
        const MAX_NODES: usize = 200;
        let max = max_nodes.min(MAX_NODES);

        let mut open: BinaryHeap<Reverse<(u32, BlockPos)>> = BinaryHeap::new();
        let mut came_from: AHashMap<BlockPos, BlockPos> = AHashMap::new();
        let mut g_score: AHashMap<BlockPos, u32> = AHashMap::new();

        g_score.insert(start, 0);
        open.push(Reverse((heuristic(start, goal), start)));

        while let Some(Reverse((_, current))) = open.pop() {
            if current == goal { return Some(reconstruct_path(came_from, current)); }
            if g_score.len() >= max { return None; }  // budget exceeded

            for neighbour in walkable_neighbours(current, world) {
                let tentative_g = g_score[&current] + move_cost(current, neighbour);
                if tentative_g < *g_score.get(&neighbour).unwrap_or(&u32::MAX) {
                    came_from.insert(neighbour, current);
                    g_score.insert(neighbour, tentative_g);
                    let f = tentative_g + heuristic(neighbour, goal);
                    open.push(Reverse((f, neighbour)));
                }
            }
        }
        None  // No path found
    }
}

/// Manhattan distance heuristic — admissible for grid movement
fn heuristic(pos: BlockPos, goal: BlockPos) -> u32 {
    ((pos.x - goal.x).abs() + (pos.y - goal.y).abs() + (pos.z - goal.z).abs()) as u32
}

/// Get walkable neighbour positions (flat + one-up + one-down, no diagonal)
fn walkable_neighbours(pos: BlockPos, world: &WorldData) -> SmallVec<[BlockPos; 6]> {
    let deltas = [(1,0,0), (-1,0,0), (0,0,1), (0,0,-1)];
    let mut result = SmallVec::new();

    for (dx, dy, dz) in deltas {
        let flat = BlockPos { x: pos.x+dx, y: pos.y+dy, z: pos.z+dz };
        // Can walk flat: current feet clear, head clear
        if world.is_walkable(flat) { result.push(flat); }
        // Can step up
        let up = BlockPos { y: flat.y + 1, ..flat };
        if world.is_walkable(up) { result.push(up); }
        // Can step down (fall one block)
        let down = BlockPos { y: flat.y - 1, ..flat };
        if world.is_walkable(down) { result.push(down); }
    }
    result
}
```

---

## ECS Schedule Setup (Bevy)

```rust
use bevy::prelude::*;

pub struct MinecraftPlugin;

impl Plugin for MinecraftPlugin {
    fn build(&self, app: &mut App) {
        app
            // Fixed 50ms tick — matches Minecraft TPS
            .add_plugins(bevy::time::TimePlugin)
            .insert_resource(Time::<Fixed>::from_hz(20.0))
            .add_systems(FixedUpdate, (
                // Input → Physics → AI — explicit ordering
                player_input_system,
                apply_player_input,
                mob_ai_system,
                physics_tick,
                // Dependency: physics runs before position sync
            ).chain());
    }
}
```

> **leonardomso rule**: `async-spawn-blocking` — run CPU-bound pathfinding off the ECS tick.
> `perf-iter-over-index` — use `par_iter_mut()` for entity physics, never manual indexing.
