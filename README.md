# layer

An async Rust library for the Telegram MTProto protocol.

[![Crates.io](https://img.shields.io/crates/v/layer-client?style=flat-square&color=fc8d62&label=layer-client)](https://crates.io/crates/layer-client)
[![docs.rs](https://img.shields.io/badge/docs.rs-layer--client-5865F2?style=flat-square)](https://docs.rs/layer-client)
[![License](https://img.shields.io/badge/license-MIT%20%7C%20Apache--2.0-blue?style=flat-square)](LICENSE-MIT)
[![TL Layer](https://img.shields.io/badge/TL%20Layer-224-8b5cf6?style=flat-square)](https://core.telegram.org/schema)
[![Telegram](https://img.shields.io/badge/chat-%40layer__chat-2CA5E0?style=flat-square&logo=telegram)](https://t.me/layer_chat)

> **Pre-production (`0.x.x`)** — APIs may change between minor versions. See [CHANGELOG](CHANGELOG.md) before upgrading.

Built by [Ankit Chaubey](https://github.com/ankit-chaubey) — *with curiosity, caffeine, and a lot of Rust compiler errors 🦀*

---

## Overview

layer is a bottom-up implementation of [Telegram MTProto](https://core.telegram.org/mtproto) in async Rust. The TL schema parser, AES-IGE cipher, Diffie-Hellman handshake, MTProto session, and update stream are all written from scratch. External deps (`tokio`, `flate2`, `getrandom`) are used where they make sense.

Built as an experiment to understand MTProto at the protocol level, not just wrap an existing SDK.

> **AI note:** Parts of the docs and boilerplate were drafted with AI assistance to reduce manual effort. Everything has been reviewed before publishing. Core library code is written and understood by the author.

---

## Crates

Most users only need `layer-client`.

| Crate | Description |
|---|---|
| [`layer-client`](./layer-client) | High-level async client: auth, messaging, media, bots |
| [`layer-tl-types`](./layer-tl-types) | Layer 224 types, functions, enums (2,329 definitions) |
| [`layer-mtproto`](./layer-mtproto) | MTProto session, DH exchange, framing, transports |
| [`layer-crypto`](./layer-crypto) | AES-IGE, RSA, SHA, Diffie-Hellman, auth key derivation |
| [`layer-tl-gen`](./layer-tl-gen) | Build-time code generator from the TL AST |
| [`layer-tl-parser`](./layer-tl-parser) | Parses `.tl` schema into an AST |

---

## Installation

```toml
[dependencies]
layer-client = "0.4.6"
tokio        = { version = "1", features = ["full"] }
```

Get your `api_id` and `api_hash` from [my.telegram.org](https://my.telegram.org).

Optional features:

```toml
layer-client = { version = "0.4.6", features = ["sqlite-session"] }  # SQLite session
layer-client = { version = "0.4.6", features = ["libsql-session"] }  # libsql / Turso
layer-client = { version = "0.4.6", features = ["html"] }            # HTML parser
layer-client = { version = "0.4.6", features = ["html5ever"] }       # html5ever parser
```

`layer-client` re-exports `layer_tl_types` as `layer_client::tl`, so you rarely need `layer-tl-types` as a direct dependency.

---

## Quick Start — Bot

```rust
use layer_client::{Client, update::Update};

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let (client, _shutdown) = Client::builder()
        .api_id(std::env::var("API_ID")?.parse()?)
        .api_hash(std::env::var("API_HASH")?)
        .session("bot.session")
        .connect()
        .await?;

    client.bot_sign_in(&std::env::var("BOT_TOKEN")?).await?;
    client.save_session().await?;

    let mut stream = client.stream_updates();
    while let Some(Update::NewMessage(msg)) = stream.next().await {
        if !msg.outgoing() {
            if let Some(peer) = msg.peer_id() {
                client.send_message_to_peer(peer.clone(), msg.text().unwrap_or("")).await?;
            }
        }
    }
    Ok(())
}
```

No trait objects, no callbacks, no `dyn Handler`. Just an async loop and pattern matching.

## Quick Start — User Account

```rust
use layer_client::{Client, SignInError};
use std::io::{self, BufRead};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let (client, _shutdown) = Client::builder()
        .api_id(12345)
        .api_hash("your_api_hash")
        .session("my.session")
        .connect()
        .await?;

    if !client.is_authorized().await? {
        let token = client.request_login_code("+1234567890").await?;
        let code  = io::stdin().lock().lines().next().unwrap()?;

        match client.sign_in(&token, &code).await {
            Ok(name) => println!("Welcome, {name}!"),
            Err(SignInError::PasswordRequired(t)) => {
                client.check_password(*t, "my_2fa_password").await?;
            }
            Err(e) => return Err(e.into()),
        }
        client.save_session().await?;
    }

    client.send_message("me", "Hello from layer!").await?;
    Ok(())
}
```

For the full API — messaging, media, keyboards, search, participants, reactions, raw invoke, transports, session backends — see **[docs.rs/layer-client](https://docs.rs/layer-client)** and the **[online guide](https://layer.ankitchaubey.in/)**.

---

## Update Stream

```rust
while let Some(update) = stream.next().await {
    match update {
        Update::NewMessage(msg)     => { /* new message */ }
        Update::MessageEdited(msg)  => { /* edited */ }
        Update::MessageDeleted(del) => { /* deleted */ }
        Update::CallbackQuery(cb)   => { /* inline button pressed */ }
        Update::InlineQuery(iq)     => { /* inline mode query */ }
        Update::UserTyping(action)  => { /* typing/uploading */ }
        Update::UserStatus(status)  => { /* online/offline */ }
        Update::Raw(raw)            => { /* anything not yet wrapped */ }
        _ => {}  // #[non_exhaustive] — always keep this arm
    }
}
```

For production bots, spawn each update into its own task to avoid blocking the loop:

```rust
let client = Arc::new(client);
let mut stream = client.stream_updates();

while let Some(update) = stream.next().await {
    let c = client.clone();
    tokio::spawn(async move {
        if let Err(e) = handle_update(update, &c).await {
            eprintln!("handler error: {e}");
        }
    });
}
```

---

## Session Backends

| Backend | Flag | Notes |
|---|---|---|
| `BinaryFileBackend` | default | Single-process bots, scripts |
| `InMemoryBackend` | default | Tests, ephemeral tasks |
| `StringSessionBackend` | default | Serverless, env-var storage |
| `SqliteBackend` | `sqlite-session` | Multi-session local apps |
| `LibSqlBackend` | `libsql-session` | Turso / distributed storage |
| Custom | — | Implement `SessionBackend` |

String sessions encode the full auth state (auth key, DC, peer cache) into a single base64 string:

```rust
let s = client.export_session_string().await?;
let (client, _) = Client::with_string_session(&s).await?;
```

---

## Feature Flags

### `layer-tl-types`

| Flag | Default | Description |
|---|:---:|---|
| `tl-api` | ✅ | Telegram API schema |
| `tl-mtproto` | ❌ | MTProto internal schema |
| `impl-debug` | ✅ | `#[derive(Debug)]` on all types |
| `impl-from-type` | ✅ | `From<types::T> for enums::E` |
| `impl-from-enum` | ✅ | `TryFrom<enums::E> for types::T` |
| `impl-serde` | ❌ | `serde::Serialize + Deserialize` |
| `name-for-id` | ❌ | `name_for_id(u32)` lookup table |

### `layer-client`

| Flag | Default | Description |
|---|:---:|---|
| `html` | ❌ | Built-in HTML parser |
| `html5ever` | ❌ | html5ever tokenizer |
| `sqlite-session` | ❌ | SQLite session backend |
| `libsql-session` | ❌ | libsql / Turso session backend |

---

## Raw API

Every Layer 224 method is accessible via `client.invoke()`, even without a high-level wrapper:

```rust
use layer_client::tl;

let req = tl::functions::bots::SetBotCommands {
    scope: tl::enums::BotCommandScope::Default(tl::types::BotCommandScopeDefault {}),
    lang_code: "en".into(),
    commands: vec![
        tl::enums::BotCommand::BotCommand(tl::types::BotCommand {
            command:     "start".into(),
            description: "Start the bot".into(),
        }),
    ],
};
client.invoke(&req).await?;

// Target a specific DC
client.invoke_on_dc(&req, 2).await?;
```

---

## Unsupported

All reachable today via `client.invoke()`.

| Feature | Notes |
|---|---|
| Secret chats | Not implemented at MTProto layer-2 |
| Voice / video calls | No signalling or media transport |
| Payments | Returns error |
| Channel creation | `channels::CreateChannel` |
| Sticker set management | `messages::GetStickerSet` |
| Poll / quiz creation | `InputMediaPoll` |
| Bot command registration | `bots::SetBotCommands` |
| IPv6 | Untested |

---

## Tests

```bash
cargo test --workspace
cargo test --workspace --all-features
```

Integration tests in `layer-client/tests/integration.rs` use `InMemoryBackend` and don't need real credentials.

---

## Community

- Channel: [t.me/layer_rs](https://t.me/layer_rs)
- Chat: [t.me/layer_chat](https://t.me/layer_chat)
- Guide: [layer.ankitchaubey.in](https://layer.ankitchaubey.in/)
- API docs: [docs.rs/layer-client](https://docs.rs/layer-client)
- Issues: [github.com/ankit-chaubey/layer/issues](https://github.com/ankit-chaubey/layer/issues)

---

## Contributing

Read [CONTRIBUTING.md](CONTRIBUTING.md) before opening a PR. Run `cargo test --workspace` and `cargo clippy --workspace` locally. Security issues: see [SECURITY.md](SECURITY.md).

---

## License

MIT or Apache-2.0, at your option. See [LICENSE-MIT](LICENSE-MIT) and [LICENSE-APACHE](LICENSE-APACHE).

---

> Ensure your usage complies with [Telegram's API Terms of Service](https://core.telegram.org/api/terms).
