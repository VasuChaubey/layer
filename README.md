# layer

An async Rust library for the Telegram MTProto protocol.

[![Crates.io](https://img.shields.io/crates/v/layer-client?style=flat-square&color=fc8d62&label=layer-client)](https://crates.io/crates/layer-client)
[![docs.rs](https://img.shields.io/badge/docs.rs-layer--client-5865F2?style=flat-square)](https://docs.rs/layer-client)
[![License](https://img.shields.io/badge/license-MIT%20%7C%20Apache--2.0-blue?style=flat-square)](LICENSE-MIT)
[![TL Layer](https://img.shields.io/badge/TL%20Layer-224-8b5cf6?style=flat-square)](https://core.telegram.org/schema)
[![Telegram](https://img.shields.io/badge/chat-%40layer__chat-2CA5E0?style=flat-square&logo=telegram)](https://t.me/layer_chat)

> **Pre-production (`0.x.x`)**: APIs may change between minor versions. See [CHANGELOG](CHANGELOG.md) before upgrading.

---

## Overview

layer is a bottom-up implementation of [Telegram MTProto](https://core.telegram.org/mtproto) in async Rust. The TL schema parser, AES-IGE cipher, Diffie-Hellman handshake, MTProto session, and update stream are all written from scratch. External deps (`tokio`, `flate2`, `getrandom`) are used where they make sense.

This is an **experiment in understanding**: built to learn how the protocol actually works, not as a production SDK. The architecture and DH session design are closely based on [grammers](https://codeberg.org/Lonami/grammers) by Lonami. Portions of the code are derived from grammers (MIT/Apache-2.0).

> **AI note:** Parts of the documentation, boilerplate, and repeated patterns in this repo were drafted with AI assistance to reduce manual effort. Everything has been reviewed by a human before publishing. The core library code is written and understood by the author.

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
layer-client = { version = "0.4.6", features = ["sqlite-session"] }   # SQLite session
layer-client = { version = "0.4.6", features = ["libsql-session"] }   # libsql / Turso
layer-client = { version = "0.4.6", features = ["html"] }             # HTML parser
layer-client = { version = "0.4.6", features = ["html5ever"] }        # html5ever parser
```

---

## Quick Start

**Bot:**

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

**User account:**

```rust
use layer_client::{Client, SignInError};

let (client, _shutdown) = Client::builder()
    .api_id(12345)
    .api_hash("your_api_hash")
    .session("my.session")
    .connect()
    .await?;

if !client.is_authorized().await? {
    let token = client.request_login_code("+1234567890").await?;
    // read code from stdin, then:
    match client.sign_in(&token, &code).await {
        Ok(_)                                 => {}
        Err(SignInError::PasswordRequired(t)) => { client.check_password(*t, "2fa_pass").await?; }
        Err(e)                                => return Err(e.into()),
    }
    client.save_session().await?;
}
```

For the full API: messaging, media, keyboards, search, participants, reactions, raw invoke, transports, session backends: see **[docs.rs/layer-client](https://docs.rs/layer-client)** and the **[online guide](https://layer.ankitchaubey.in/)**.

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

## Unsupported

All of these are reachable today via `client.invoke()` with raw TL types.

| Feature | Notes |
|---|---|
| Secret chats | Not implemented at MTProto layer-2 |
| Voice / video calls | No signalling or media transport |
| Payments | Returns error |
| Channel creation | Use `channels::CreateChannel` |
| Sticker set management | Use `messages::GetStickerSet` |
| Poll / quiz creation | Use `InputMediaPoll` |
| Bot command registration | Use `bots::SetBotCommands` |
| IPv6 | Untested |

---

## Tests

```bash
cargo test --workspace
cargo test --workspace --all-features
```

Integration tests in `layer-client/tests/integration.rs` use `InMemoryBackend` and don't need real credentials.

---

## Contributing

Read [CONTRIBUTING.md](CONTRIBUTING.md) before opening a PR. Run `cargo test --workspace` and `cargo clippy --workspace` locally. Security issues: see [SECURITY.md](SECURITY.md): don't open a public issue.

---

## Acknowledgements

- [Lonami](https://codeberg.org/Lonami) / [grammers](https://codeberg.org/Lonami/grammers): architecture, DH session design, SRP 2FA math, and session handling are based on this library. Derived code is used under MIT/Apache-2.0.
- [Telegram](https://core.telegram.org/mtproto) for the MTProto spec and public TL schema.
- `tokio`, `flate2`, `getrandom`, `sha2`, `socket2`.

---

## License

MIT or Apache-2.0, at your option. See [LICENSE-MIT](LICENSE-MIT) and [LICENSE-APACHE](LICENSE-APACHE).

Contributions are dual-licensed under the same terms.

---

> Ensure your usage complies with [Telegram's API Terms of Service](https://core.telegram.org/api/terms). Spam, mass scraping, or automating normal user accounts may result in bans.
