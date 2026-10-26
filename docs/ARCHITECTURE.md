# nyx-kv — Architecture

## Overview

nyx-kv is a single-binary, in-memory key-value store written in Nyx. It implements the RESP2 protocol and is compatible with any Redis client.

```
Client (redis-cli / redis-py / ioredis)
    │  TCP :6380  (or TLS)
    ▼
nyx-kv main.nx
    │  one goroutine per connection (spawn)
    ▼
commands.nx  ──► store.nx  ──► runtime persist.c (.ndb snapshots)
    │
    ├── auth.nx      (token management, namespace isolation)
    ├── pubsub.nx    (channel subscriptions, fan-out delivery)
    └── limits.nx    (rate limiting, per-IP sliding window)
```

## Components

### main.nx

Entry point. Parses CLI flags (`--public`, `--tls`, `--rate-limit`, `--no-rate-limit`, `--port`), initialises the store, auth, and limits subsystems, then enters the accept loop. Each accepted connection is handed off to `kv_worker` (plain TCP) or `kv_tls_worker` (TLS) via `spawn`.

### store.nx

Holds the global `g_store: Map` and type-tag map `g_types: Map`. Implements all data-type operations (strings, lists, sets, hashes) plus TTL tracking via `g_ttl: Map`. All mutations are protected by a single global mutex `g_store_lock`.

Key design decisions:
- Lists are stored as newline-delimited strings packed in the Map (avoids nested GC complexity).
- Sets are stored as maps from member → "1".
- Hashes are stored as maps with `key\x01field` composite keys.

### commands.nx

Dispatcher. Parses the RESP array from the client, routes to the correct store operation, applies public-mode restrictions, enforces plan limits, and writes the RESP response back.

### auth.nx

Token lifecycle management. Tokens are 32-byte random values encoded as 64-char hex strings. Stored internally as `token → "user_id:plan:expires_at"`. Expiry is checked at AUTH time using `time_ms() / 1000`.

Admin commands (TOKEN_CREATE, TOKEN_REVOKE, TOKEN_LIST) are restricted to connections from `127.0.0.1`.

### pubsub.nx

Fan-out pub/sub. `g_subscribers: Map` maps channel → list of file descriptors. PUBLISH iterates the list and writes RESP message frames to each subscriber fd. Subscribers that have disconnected are pruned on the next PUBLISH cycle.

### limits.nx

Per-IP rate limiting using a sliding window of 1 second. `g_rate_ts: Map` (ip → last_reset_us) and `g_rate_count: Map` (ip → count). The limit is set at startup (default: 100 req/s free, 10000 req/s pro, unlimited enterprise).

## Data Persistence

Binary `.ndb` snapshots are written by `runtime/persist.c`:
- Magic header + version byte
- Sorted key list with type tags, values, TTLs
- CRC32 checksum at end

SAVE is synchronous. BGSAVE spawns a goroutine that serialises a snapshot without blocking the accept loop.

On startup, if `dump.ndb` exists it is loaded automatically before the first connection is accepted.

## TLS

When `--tls` is passed, `tls_server_init(cert, key, port)` is called instead of a plain TCP bind. Each connection uses `tls_accept()` → `tls_read_line()` / `tls_write_conn()` from `runtime/tls.c` (OpenSSL wrapper). The RESP parser is duplicated for TLS (`resp_read_tls`) since the transport read functions differ.

## Concurrency Model

One goroutine per client connection (M:N work-stealing scheduler). The store is protected by a single coarse mutex. This is sufficient because most operations are O(1) and the bottleneck is network I/O, not lock contention.

Pub/Sub fan-out and BGSAVE both use `spawn` goroutines with no additional synchronisation beyond the store lock.
