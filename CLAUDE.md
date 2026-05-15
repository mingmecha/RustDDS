# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

### Build & Check
```bash
cargo build                          # default features (no security)
cargo build --features security      # with DDS Security
cargo check                          # fast type-check only
cargo doc                            # build documentation
```

### Format & Lint
Formatting and linting require the **nightly** toolchain:
```bash
cargo +nightly fmt                                              # format in-place
cargo +nightly fmt -- --check                                  # check only (CI)
cargo +nightly clippy --tests --examples                       # lint default features
cargo +nightly clippy --tests --examples --all-features        # lint all features
```
The `prepare-for-commit.sh` script runs all three of the above in sequence.

### Tests
Tests **must** run single-threaded due to shared network resources:
```bash
cargo test -- --test-threads=1                   # default features
cargo test --features security -- --test-threads=1  # with security
```
Run a single test:
```bash
cargo test <test_name> -- --test-threads=1
cargo test --test mio_08_pub_sub_test -- --test-threads=1   # specific integration test file
```

## Architecture

### Threading Model
`DomainParticipant::new()` spawns two long-running background threads, communicating via `mio_extras::channel` (mio-pollable channels):

1. **DP Event Loop** (`src/rtps/dp_event_loop.rs`) — handles all RTPS network I/O: receives UDP packets, processes RTPS messages/submessages, drives local `Reader` and `Writer` instances, and sends outgoing data.
2. **Discovery thread** (`src/discovery/discovery.rs`) — runs SPDP (participant discovery) and SEDP (endpoint discovery), maintains the `DiscoveryDB`, and notifies the event loop when remote endpoints appear/disappear.

The `DomainParticipant` itself is a handle that communicates with these threads through channels. `DataReader`/`DataWriter` objects are also handles; data moves through `DDSCache` (`src/structure/dds_cache.rs`), a shared `Arc<RwLock<DDSCache>>` that both the event loop and the DDS API layer access.

### Module Map
```
src/
  dds/           Public DDS API: DomainParticipant, Publisher, Subscriber,
                   Topic, DataReader, DataWriter, QoS, error types
  rtps/          RTPS protocol: message receiver, reader/writer state machines,
                   reader/writer proxy objects, event loop
  messages/      RTPS wire format: message headers, all submessage types
  structure/     Fundamental types: GUID, Locator, SequenceNumber,
                   CacheChange, DDSCache, Time, Duration
  network/       UDP transport: UDPListener, UDPSender
  discovery/     SPDP + SEDP: participant/endpoint announcement and discovery DB
  serialization/ CDR serializer/deserializer adapters wrapping the cdr-encoding crate
  security/      DDS Security (feature-gated): authentication, access control,
                   cryptographic plugins, X.509 certificates
  no_security/   Stub types used when the security feature is disabled,
                   so callers don't need #[cfg] everywhere
```

### DDS Topic Variants: `with_key` vs `no_key`
Every public-facing DDS type has two variants, found in `src/dds/with_key/` and `src/dds/no_key/`. With_key topics carry multiple keyed *instances*; no_key topics have a single instance. The `Keyed` trait (in `src/dds/key.rs`) must be implemented for payload types used on with_key topics.

### Serialization
RustDDS does not generate code for specific message types. Instead, `DataReader<D, SA>` and `DataWriter<D, SA>` are generic over both the payload type `D` (must be `serde::Serialize`/`DeserializeOwned`) and a serializer adapter `SA`. The default adapter is `CDRSerializerAdapter`/`CDRDeserializerAdapter` backed by the external `cdr-encoding` crate.

### Polling / Async
Three non-exclusive I/O models are supported:
- **mio-0.6** — `DataReader` implements `mio_06::event::Evented`
- **mio-0.8** — `DataReader` implements `mio_08::event::Source`
- **async** — `DataReader`/`DataWriter` can be converted to `futures::stream::Stream` via `async_sample_stream()`

See examples `shapes_demo`, `shapes_demo_mio_08`, and `async_shapes_demo` for working usage of each.

### Error Handling Conventions
Errors use `thiserror`-derived enums (`ReadError`, `WriteError<D>`, `CreateError`, `WaitError`) in `src/dds/result.rs`. Companion macros (e.g. `read_error_internal!`, `create_error_poisoned!`) log with `log::error!` and immediately return the `Err` variant — use these macros instead of constructing error variants directly.

### Feature Flags
- `security` — enables DDS Security (OMG spec v1.1): adds authentication, access control, and cryptographic plugins plus many new dependencies (OpenSSL, X.509, ASN.1 crates). The `no_security` module provides stub types when this feature is absent.
- `build_openssl` — when used together with `security`, builds OpenSSL from source instead of linking the system copy.

## Key Conventions
- **rustfmt** uses 2-space indentation (`tab_spaces = 2`) with nightly features — always format with `cargo +nightly fmt`.
- The `ros2` module inside RustDDS is deprecated; the separate `ros2-client` crate should be used for ROS2 integration.
- MSRV is Rust 1.82.0 (enforced in CI); avoid language features newer than that unless the MSRV is explicitly bumped.
- Tests involving network sockets (most integration tests) are sensitive to concurrency — always pass `--test-threads=1`.
