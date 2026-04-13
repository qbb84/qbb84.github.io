---
layout: post
author: "qbb84"
tags: ["Rust", "Minecraft", "Networking", "Protocols", "Game Development"]
---

*This is a technical dev log covering Nebula — a Minecraft server implementation written from scratch in Rust, including the full authentication handshake, packet codec, and a custom proc-macro crate for packet serialisation.*

## Why Rust, and Why From Scratch

Most Minecraft server development happens on top of Spigot or Paper — abstractions that handle the protocol layer entirely. You write plugins, respond to events, and never think about what's happening on the wire. That's the right approach for building games. But I wanted to understand the protocol at a level that Spigot doesn't expose.

Rust was the natural choice. The Minecraft protocol is fundamentally a binary serialisation problem over a TCP stream — exactly the kind of problem Rust handles elegantly, with zero-cost abstractions, explicit memory ownership, and no garbage collector introducing latency. Writing a game server in a language that competes with C for raw throughput is the right call when you're thinking about hundreds of thousands of concurrent connections.

## The Minecraft Protocol

Before writing a line of code, I spent time with [wiki.vg](https://wiki.vg) — the community-maintained Minecraft protocol documentation. The protocol has four states: **Handshaking**, **Status**, **Login**, and **Play**. Each state defines a set of packets, each with a specific format.

Packets are structured as:

```
[Length: VarInt] [Packet ID: VarInt] [Data: ...]
```

A VarInt is a variable-length integer encoding — the same format used in Protocol Buffers. Each byte uses 7 bits for data and 1 bit as a continuation flag. Small numbers take 1 byte, larger ones take up to 5.

Reading and writing VarInts correctly is the first thing you implement, and the first thing that breaks when you get it wrong.

```rust
pub fn read_varint(buf: &mut Cursor<&[u8]>) -> Result<i32> {
    let mut result = 0i32;
    let mut shift = 0;
    loop {
        let byte = buf.read_u8()?;
        result |= ((byte & 0x7F) as i32) << shift;
        if byte & 0x80 == 0 {
            break;
        }
        shift += 7;
        if shift >= 35 {
            return Err(anyhow!("VarInt too long"));
        }
    }
    Ok(result)
}
```

## The Authentication Handshake

This is the part that separates a toy implementation from something real. Online mode Minecraft uses a three-step authentication process involving RSA encryption, AES stream cipher, and Mojang's session server.

### Step 1 — Encryption Request

The server generates an RSA key pair and sends the client a public key alongside a random 4-byte verify token:

```rust
let rsa = Rsa::generate(1024)?;
let public_key_der = rsa.public_key_to_der()?;

// Send Encryption Request packet
// [Public Key Length: VarInt] [Public Key: Bytes]
// [Verify Token Length: VarInt] [Verify Token: Bytes]
```

### Step 2 — Encryption Response

The client responds with two values, both RSA-encrypted with the server's public key:

- A 16-byte shared secret — this becomes the AES key
- The verify token, echoed back to prove the client decrypted it correctly

```rust
let shared_secret = rsa.private_decrypt(&encrypted_secret, Padding::PKCS1)?;
let decrypted_token = rsa.private_decrypt(&encrypted_token, Padding::PKCS1)?;

assert_eq!(decrypted_token, verify_token);
```

### Step 3 — Session Authentication

Before enabling encryption, the server must verify the client is who they claim to be. This involves computing a SHA1 hash — called the "server hash" — derived from the server ID (an empty string in modern versions), the shared secret, and the public key:

```rust
let server_hash = compute_server_hash("", &shared_secret, &public_key_der);
```

The server hash has a peculiarity: it can be negative. Java's `BigInteger.toString(16)` produces a two's complement hex representation, which means negative values are prefixed with a minus sign. Rust's standard library doesn't do this natively — getting this wrong means authentication will always fail and it took a while to track down.

```rust
pub fn compute_server_hash(server_id: &str, shared_secret: &[u8], public_key: &[u8]) -> String {
    let mut hasher = Sha1::new();
    hasher.update(server_id.as_bytes());
    hasher.update(shared_secret);
    hasher.update(public_key);
    let hash = hasher.finalize();

    // Java BigInteger two's complement hex formatting
    let bigint = BigInt::from_signed_bytes_be(&hash);
    format!("{:x}", bigint)
}
```

The server then calls Mojang's session server to verify the player:

```
GET https://sessionserver.mojang.com/session/minecraft/hasJoined
    ?username=<name>&serverId=<hash>
```

A 200 response with a profile JSON means the player is authenticated. Anything else and you disconnect them.

### Step 4 — AES-CFB8

Once authentication passes, all subsequent communication is encrypted using AES in CFB8 mode with the shared secret as both the key and the IV:

```rust
use aes::Aes128;
use cfb8::Cfb8;
use cfb8::cipher::{AsyncStreamCipher, NewCipher};

type AesCfb8 = Cfb8<Aes128>;

let encryptor = AesCfb8::new_from_slices(&shared_secret, &shared_secret)?;
let decryptor = AesCfb8::new_from_slices(&shared_secret, &shared_secret)?;
```

Every byte sent and received after this point passes through the cipher. The stream is symmetric — both sides encrypt with the same key and IV, decrypting on receipt.

## The Packet Macro System

Implementing packets manually is tedious and error-prone. Each packet is just a struct with a specific serialisation format — read fields in order, write fields in order. This is exactly the kind of boilerplate that proc-macros eliminate.

I built `packet_macro` and `packet_codec` as separate crates. The macro derives `Encode` and `Decode` traits for any struct annotated with `#[derive(Packet)]`:

```rust
#[derive(Packet)]
pub struct LoginSuccess {
    pub uuid: Uuid,
    pub username: String,
    pub properties: Vec<Property>,
}
```

The macro expands this to:

```rust
impl Encode for LoginSuccess {
    fn encode(&self, buf: &mut BytesMut) {
        self.uuid.encode(buf);
        self.username.encode(buf);
        self.properties.encode(buf);
    }
}

impl Decode for LoginSuccess {
    fn decode(buf: &mut Cursor<&[u8]>) -> Result<Self> {
        Ok(Self {
            uuid: Uuid::decode(buf)?,
            username: String::decode(buf)?,
            properties: Vec::<Property>::decode(buf)?,
        })
    }
}
```

Adding a new packet is now a single struct definition. The macro handles everything else. This was one of the more satisfying parts of the project — the first time a new packet compiled and worked correctly without writing any serialisation code manually.

## Async Architecture

The server runs on Tokio — Rust's async runtime. Each client connection spawns a task that owns a read half and a write half of the TCP stream:

```rust
let (reader, writer) = stream.split();

tokio::spawn(async move {
    handle_client(reader, writer, server_state).await;
});
```

Tokio's task scheduler handles thousands of concurrent connections efficiently across a thread pool sized to the machine's CPU count. There's no thread-per-connection overhead — the async model means a connection waiting on a network read consumes virtually no resources.

## Dependencies

The `Cargo.toml` tells the full story of what's involved:

```toml
[dependencies]
tokio       = { version = "1", features = ["full"] }
serde       = { version = "1.0", features = ["derive"] }
bytes       = { version = "1", features = ["serde"] }
rsa         = "0.9.7"
rand        = "0.8.5"
sha1        = "0.10.6"
aes         = "0.8.4"
cfb8        = "0.8.1"
cipher      = "0.4.4"
num-bigint  = "0.4.6"
reqwest     = { version = "0.12.9", features = ["json"] }
packet_macro = { path = "../packet_macro" }
packet_codec = { path = "../packet_codec" }
```

Each dependency has a specific purpose. Nothing is included without a reason.

## What I Learned

The Minecraft protocol is well-documented but unforgiving. The SHA1 two's complement formatting, the CFB8 IV being the same as the key, the precise ordering of fields in every packet — any deviation produces a client disconnect with no useful error message. You learn to read packet captures.

More broadly: implementing a protocol from scratch teaches you things that working above an abstraction layer never does. After Nebula, reading Spigot's NMS internals feels familiar rather than opaque. The abstraction makes more sense when you understand what it's abstracting.

The packet macro system was the architectural highlight. Building a proc-macro that generates correct serialisation code for arbitrary structs — and then watching it just work across every packet type — is the kind of leverage that makes Rust genuinely exciting to work in.

## What's Next

The current build covers Handshaking, Status, and Login states in full. Play state packets follow the same patterns — the macro system makes adding them straightforward. The foundation is the hard part. The rest is implementation.

[View Repo](https://github.com/qbb84/Nebula)
