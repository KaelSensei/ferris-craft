---
name: mc-06-networking
description: >
  Layer 2 — Design Choices (WHAT). Async Minecraft server architecture: tick loop
  separation, packet I/O, protocol state machine (Handshake → Status → Login → Play),
  compression, encryption, and the channel patterns that keep I/O off the tick thread.
layer: 2
triggers:
  - server, protocol, packet, tokio, TPS, networking, async server, handshake
  - login, play state, packet parser, compression, encryption, connection
---

# mc-06 — Async Server Networking

> **Core Question**: *How do you handle hundreds of connections without ever blocking
> the 50ms game tick?*
>
> Answer: **Strict thread boundary** — I/O threads feed channels, tick thread drains them.
> The tick thread must NEVER do I/O, DNS, or blocking file operations.

---

## Architecture Overview

```
                     ┌───────────────────────────┐
                     │   Tick Thread (20 TPS)     │
                     │   Fixed 50ms interval       │
                     │   ECS systems run here      │
                     │   Drains packet_rx channel  │
                     └──────────────┬─────────────┘
                                    │ Arc<World>
                     ┌──────────────▼─────────────┐
                     │   Channel Bridge            │
                     │   packet_tx → packet_rx     │  ← bounded mpsc
                     │   outbound_tx per player    │
                     └──────────────┬─────────────┘
                                    │
        ┌───────────────────────────┼───────────────────────────┐
        ▼                           ▼                           ▼
┌──────────────┐          ┌──────────────┐           ┌──────────────┐
│ Connection 1  │          │ Connection 2  │           │ Connection N  │
│ tokio::task   │          │ tokio::task   │           │ tokio::task   │
│ read packets  │          │ read packets  │           │ read packets  │
│ write packets │          │ write packets │           │ write packets │
└──────────────┘          └──────────────┘           └──────────────┘
```

---

## Rule: `mc-net-tick-separation` — Strict tick/IO thread separation

```rust
use tokio::net::TcpListener;
use tokio::sync::mpsc;
use std::time::Duration;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    // Bounded channel: backpressure prevents packet flood from overwhelming tick
    let (packet_tx, mut packet_rx) = mpsc::channel::<IncomingPacket>(4096);

    // ── TICK LOOP (runs at exactly 20 TPS) ──────────────────────────────────
    tokio::spawn(async move {
        let mut interval = tokio::time::interval(Duration::from_millis(50));
        interval.set_missed_tick_behavior(tokio::time::MissedTickBehavior::Delay);
        let mut world = World::new();
        let mut tick_count = 0u64;

        loop {
            let tick_start = interval.tick().await;

            // Drain ALL pending packets (non-blocking: stop when channel empty)
            // Cap at N packets per tick to bound work even under flood
            let mut packets_this_tick = 0usize;
            while packets_this_tick < 1000 {
                match packet_rx.try_recv() {
                    Ok(pkt)  => { world.apply_packet(pkt); packets_this_tick += 1; }
                    Err(_)   => break,  // channel empty or lagged
                }
            }

            // ECS tick — all game logic here
            world.tick(tick_count);
            tick_count += 1;

            // Warn if tick is overrunning budget
            let elapsed = tick_start.elapsed();
            if elapsed > Duration::from_millis(45) {
                tracing::warn!("Tick {tick_count} overran budget: {elapsed:?}");
            }
        }
    });

    // ── ACCEPT LOOP (separate from tick) ────────────────────────────────────
    let listener = TcpListener::bind("0.0.0.0:25565").await?;
    tracing::info!("Listening on :25565");

    loop {
        let (stream, addr) = listener.accept().await?;
        let tx = packet_tx.clone();
        // Spawn isolated connection handler — doesn't block accept loop
        tokio::spawn(async move {
            if let Err(e) = handle_connection(stream, addr, tx).await {
                tracing::debug!("Connection {addr} closed: {e}");
            }
        });
    }
}
```

---

## Rule: `mc-net-protocol-state` — Protocol state machine

The Minecraft protocol has 4 states. Violating state transitions is an easy security bug.

```rust
/// Minecraft protocol connection state machine.
/// State transitions are one-way: Handshake → Status/Login → Play
#[derive(Debug, PartialEq, Eq, Clone, Copy)]
pub enum ProtocolState {
    Handshake,  // Initial state — determines next state from client intent
    Status,     // Server list ping — terminates after status response
    Login,      // Authentication, compression negotiation
    Play,       // In-game — all game packets
}

/// Per-connection handler — owns the TCP stream
pub struct ConnectionHandler {
    state: ProtocolState,
    reader: BufReader<OwnedReadHalf>,
    writer: BufWriter<OwnedWriteHalf>,
    compression_threshold: Option<i32>,  // None = no compression
    // Encryption keys added after Login Encryption Response
}

impl ConnectionHandler {
    pub async fn run(mut self, packet_tx: Sender<IncomingPacket>) -> anyhow::Result<()> {
        loop {
            match self.state {
                ProtocolState::Handshake => self.handle_handshake().await?,
                ProtocolState::Status    => { self.handle_status().await?; break; }
                ProtocolState::Login     => self.handle_login(&packet_tx).await?,
                ProtocolState::Play      => self.handle_play(&packet_tx).await?,
            }
        }
        Ok(())
    }

    async fn handle_handshake(&mut self) -> anyhow::Result<()> {
        let packet = self.read_packet().await?;
        // Packet 0x00: Handshake
        let next_state = packet.read_varint()?;
        self.state = match next_state {
            1 => ProtocolState::Status,
            2 => ProtocolState::Login,
            _ => anyhow::bail!("invalid next_state {next_state}"),
        };
        Ok(())
    }
}
```

---

## Rule: `mc-net-varint` — VarInt encoding (Minecraft packet framing)

Every Minecraft packet is prefixed with a VarInt length. This is the lowest-level
primitive of the protocol — get it right or nothing works.

```rust
use bytes::{Buf, BufMut, BytesMut};
use anyhow::{anyhow, Result};

/// Decode a VarInt from a buffer. Returns (value, bytes_consumed).
/// VarInt: 1–5 bytes, 7 bits per byte, LSB first, MSB = "more bytes follow"
pub fn read_varint(buf: &mut impl Buf) -> Result<i32> {
    let mut result = 0i32;
    let mut shift = 0u32;

    loop {
        if !buf.has_remaining() {
            anyhow::bail!("buffer exhausted reading VarInt");
        }
        let byte = buf.get_u8();
        result |= ((byte & 0x7F) as i32) << shift;

        if byte & 0x80 == 0 { return Ok(result); }

        shift += 7;
        if shift >= 35 {
            anyhow::bail!("VarInt too long (> 5 bytes)");
        }
    }
}

/// Encode a VarInt into a buffer.
pub fn write_varint(buf: &mut impl BufMut, mut value: i32) {
    loop {
        let byte = (value & 0x7F) as u8;
        value >>= 7;
        // Cast to u32 to handle sign bit correctly in check
        if value != 0 {
            buf.put_u8(byte | 0x80);
        } else {
            buf.put_u8(byte);
            break;
        }
    }
}

/// Calculate bytes needed to encode a VarInt
pub fn varint_size(value: i32) -> usize {
    match value {
        0..=127         => 1,
        128..=16383     => 2,
        16384..=2097151 => 3,
        _               => 5,
    }
}
```

---

## Rule: `mc-net-outbound-channel` — Per-player outbound channel for packet sending

```rust
/// Each player gets their own bounded outbound channel.
/// The tick thread sends to it; the connection task drains it.
pub struct PlayerConnection {
    pub id: EntityId,
    pub outbound: mpsc::Sender<OutboundPacket>,
}

/// Send a packet to a player — non-blocking from tick thread
pub fn send_packet(player: &PlayerConnection, packet: OutboundPacket) {
    // try_send: never blocks tick thread
    // If channel is full (player lagging), drop packet (configurable)
    match player.outbound.try_send(packet) {
        Ok(()) => {}
        Err(mpsc::error::TrySendError::Full(_)) => {
            // Player is too far behind — server can kick or skip packets
            tracing::debug!("Player {:?} outbound channel full — dropping packet", player.id);
        }
        Err(mpsc::error::TrySendError::Closed(_)) => {
            // Player disconnected — connection task is gone
        }
    }
}
```

---

## Recommended Cargo.toml for Networking

```toml
[dependencies]
tokio = { version = "1", features = ["full"] }
bytes = "1"          # Zero-copy packet buffers
anyhow = "1"         # Error handling in connection handlers
thiserror = "1"      # Protocol-level error types
tracing = "0.1"      # Structured logging (replace println!)
tracing-subscriber = { version = "0.3", features = ["env-filter"] }
aes = "0.8"          # AES-128-CFB8 for Minecraft encryption
cfb8 = "0.8"         # CFB8 stream cipher mode
flate2 = "1"         # Zlib compression for packets
```
