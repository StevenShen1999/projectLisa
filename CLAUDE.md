# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Our Tango** — A Rust implementation of an E2E encrypted messaging app. The specs in this repo (`openapi.yaml`, `asyncapi.yaml`) are the source of truth for the API being built.

## Rust Standards

- Follow the [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/).
- Prefer `thiserror` for library error types and `anyhow` for application-level error propagation.
- Use `clippy` with no warnings tolerated (`#![deny(clippy::all)]` or enforce via CI).
- All public items must have doc comments (`///`).
- Prefer `impl Trait` in argument position and concrete types in return position unless flexibility is needed.
- Use `#[must_use]` on functions where ignoring the return value is almost certainly a bug.
- Avoid `unwrap()` and `expect()` in library/production code; propagate errors with `?`.
- Prefer strong types over primitive obsession — wrap UUIDs, tokens, and domain identifiers in newtypes.

## Commands

```bash
cargo build              # Build
cargo run                # Run
cargo test               # Run all tests
cargo test <name>        # Run a single test by name (substring match)
cargo clippy -- -D warnings  # Lint (treat warnings as errors)
cargo fmt                # Format code
cargo fmt -- --check     # Check formatting without modifying
```

## Testing

- Unit tests live in a `#[cfg(test)]` module at the bottom of the file they test.
- Integration tests live in `tests/` at the crate root.
- Use `tokio::test` for async tests.
- Aim for full coverage of error paths, not just happy paths.
- Use `mockall` or similar for mocking traits at unit-test boundaries.
- Database/service integration tests should use a real instance (e.g. via `testcontainers`) rather than mocks where practical.

| File | Covers | Spec |
|------|--------|------|
| `openapi.yaml` | REST API (Auth, Users, Keys, Chats, Media, Notifications) | OpenAPI 3.0.3 |
| `asyncapi.yaml` | WebSocket events (messaging, typing, presence, receipts) | AsyncAPI 3.0.0 |

## Validation

```bash
# Validate OpenAPI spec
npx @redocly/cli lint openapi.yaml

# Validate AsyncAPI spec
npx -y @asyncapi/cli validate asyncapi.yaml
```

## Previewing Docs

```bash
# OpenAPI — Redoc
npx @redocly/cli preview-docs openapi.yaml

# AsyncAPI — local studio
npx -y @asyncapi/cli start studio

# AsyncAPI — generate static HTML
npx -y @asyncapi/cli generate fromTemplate asyncapi.yaml @asyncapi/html-template -o docs/asyncapi
```

## Architecture

### E2E Encryption Model (Signal Protocol)

- On registration, clients submit an `identity_public_key` (`POST /api/auth/register`).
- Before messaging, the sender fetches the recipient's key bundle (`GET /api/keys/bundle/{userId}`), which includes the identity key, a signed pre-key, and an optional one-time pre-key (consumed on fetch).
- The client establishes a Double Ratchet session locally. All message content (`ciphertext`) is Base64-encoded and opaque to the server.
- Clients must replenish one-time pre-keys proactively (`POST /api/keys/pre-keys`) when the count drops low (`GET /api/keys/pre-keys/count`).

### REST API Domain Groups

- **Auth** — register/login/refresh/logout + account deletion; uses JWT bearer tokens.
- **Users** — profile CRUD, user search, contacts list.
- **Keys** — key bundle upload/fetch, pre-key replenishment.
- **Chats** — conversation list, paginated encrypted message history (`before` cursor), message deletion.
- **Media** — encrypted file upload (multipart) and download (octet-stream).
- **Notifications** — push device registration and per-user preferences.

### WebSocket Transport

Connect at `ws(s)://host/ws?token=<jwt>`. All frames use a `WsEnvelope` wrapper:

```json
{ "event": "<event-type>", "payload": { ... }, "request_id": "<optional>" }
```

Event types:
- Client → Server: `message:send`, `message:read`, `typing:start`, `typing:stop`, `presence:update`
- Server → Client: `message:new`, `message:delivered`, `message:read`, `typing:update`, `presence:update`, `session:conflict`

`client_message_id` serves as an idempotency key for sent messages; the server echoes it back in `message:new` for local correlation.

### Key Design Decisions

- All IDs are UUIDs.
- Pagination on message history uses a `before` (ISO 8601 timestamp) cursor with a `has_more` flag, not page numbers.
- `session:conflict` is fired server-side when a second device authenticates with the same account — clients should handle this gracefully.
- Media files are always stored encrypted; the server never sees plaintext content.
