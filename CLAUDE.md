# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Our Tango** — A Rust implementation of a simple messaging app. The specs in this repo (`openapi.yaml`, `asyncapi.yaml`) are the source of truth for the API being built.

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
| `openapi.yaml` | REST API (Auth, Contacts, Message History) | OpenAPI 3.0.3 |
| `asyncapi.yaml` | WebSocket events (send/receive messages) | AsyncAPI 3.0.0 |

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

### REST API

- **Auth** — register + login; both return a JWT access token.
- **Users** — paginated contacts list with latest message per contact; paginated message history with a specific user.

### WebSocket Transport

Connect at `ws(s)://host/ws?token=<jwt>`. All frames use a `WsEnvelope` wrapper:

```json
{ "event": "<event-type>", "payload": { ... } }
```

Event types:
- Client → Server: `message:send`
- Server → Client: `message:new`

`client_message_id` is set by the client on `message:send` and echoed back in `message:new` for local message correlation.

### Key Design Decisions

- All IDs are UUIDs.
- Pagination uses a `before` (ISO 8601 timestamp) cursor with a `has_more` flag, not page numbers.
- Messages are stored and transmitted as plain text.
