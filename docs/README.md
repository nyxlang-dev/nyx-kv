# nyx-kv — Redis-Compatible Key-Value Store

A high-performance, Redis-compatible key-value store built entirely in [Nyx](https://nyxlang.com).

- **Protocol**: RESP (Redis Serialization Protocol) — works with `redis-cli`, Python `redis`, Node `ioredis`, Go `go-redis`, etc.
- **Port**: 6380 (default)
- **Performance**: 6.76M SET/s, 21.57M GET/s (in-process), 161K ops/s pipelined over TCP
- **Persistence**: Binary `.ndb` snapshots with background saving
- **Multi-tenant**: AUTH tokens, namespace isolation, per-user rate limits

---

## Quick Start (30 seconds)

```bash
# Start server (private mode — no limits)
./nyx-kv

# Start server (public mode — free tier limits)
./nyx-kv --public

# Connect
redis-cli -p 6380

# Try it
SET name "Nyx"
GET name
# → "Nyx"
```

## Connecting from other languages

**Python**
```python
import redis
r = redis.Redis(host='localhost', port=6380, decode_responses=True)
r.set('key', 'value')
print(r.get('key'))  # → "value"
```

**Node.js**
```javascript
const Redis = require('ioredis');
const r = new Redis(6380);
await r.set('key', 'value');
console.log(await r.get('key'));  // → "value"
```

**Go**
```go
import "github.com/redis/go-redis/v9"
rdb := redis.NewClient(&redis.Options{Addr: "localhost:6380"})
rdb.Set(ctx, "key", "value", 0)
val, _ := rdb.Get(ctx, "key").Result()
fmt.Println(val)  // → "value"
```

**redis-cli**
```bash
redis-cli -h localhost -p 6380
```

---

## Modes

### Private Mode (default)

```bash
./nyx-kv
```

No limits, no authentication required. All commands available. Suitable for local development or internal use.

### Public Mode

```bash
./nyx-kv --public
```

Free tier limits apply:
- 100 requests/second per IP
- 1,000 keys maximum
- 100KB max value size
- 72-hour TTL enforced on all keys
- `FLUSHDB`, `CONFIG`, `SAVE`, `BGSAVE` disabled

Authenticated users (via `AUTH`) get limits based on their plan (free/pro/enterprise).

---

## Authentication (Public Mode)

In public mode, users can authenticate with a token to get their own isolated namespace.

```
AUTH <token>
# → +OK (if valid)
# → -ERR invalid token (if invalid)

WHOAMI
# → "user_id (plan)"

# Admin only (localhost):
TOKEN_CREATE <user_id> <plan> [ttl_seconds]
# → 64-char hex token
# Plans: free, pro, enterprise
# ttl_seconds: 0 = no expiry (default), >0 = expires after N seconds

TOKEN_REVOKE <token>
# → OK

TOKEN_LIST
# → ["alice:pro:0", "bob:free:1743638400", ...]
```

After `AUTH`, all keys are transparently prefixed with `user_id::`. Two authenticated users cannot see each other's keys.

---

## TLS

```bash
# Start with TLS (cert paths from env vars or defaults)
./nyx-kv --tls

# Combined with public mode
./nyx-kv --public --tls

# Custom cert paths
TLS_CERT=/path/to/cert.pem TLS_KEY=/path/to/key.pem ./nyx-kv --tls
```

Connect with `redis-cli --tls` or any Redis client with SSL support. See [COMMANDS.md#TLS](COMMANDS.md#tls-mode) for details.

---

## Command Reference

→ See [COMMANDS.md](COMMANDS.md) for the full reference of all 52 commands with syntax, examples, and return types.

---

## Persistence

nyx-kv uses binary `.ndb` (Nyx Database) snapshots for persistence.

- **Background saving**: Automatic periodic snapshots
- **Shutdown saving**: Data saved on SIGTERM/SIGINT
- **Restore**: Automatic on startup if `dump.ndb` exists

```
SAVE        # Synchronous save (private mode only)
BGSAVE      # Background save (private mode only)
LASTSAVE    # Timestamp of last save (unix seconds)
```

The `.ndb` format stores strings, lists, sets, hashes, TTLs, and auth tokens in a compact binary format with CRC32 integrity checks.

---

## Data Types

| Type | Description | Commands |
|------|-------------|----------|
| **String** | Binary-safe string values | SET, GET, INCR, APPEND, ... |
| **List** | Ordered collection (doubly-linked) | LPUSH, RPUSH, LPOP, RPOP, LRANGE, ... |
| **Set** | Unordered collection of unique strings | SADD, SREM, SISMEMBER, SMEMBERS, ... |
| **Hash** | Field-value maps within a key | HSET, HGET, HDEL, HGETALL, ... |

---

## Pub/Sub

nyx-kv supports publish/subscribe messaging:

```
# Terminal 1 — subscriber
SUBSCRIBE news
# (blocks, waiting for messages)

# Terminal 2 — publisher
PUBLISH news "Breaking: Nyx v1.0 released"
# → :1 (1 subscriber received it)

# Terminal 1 receives:
# *3
# message
# news
# Breaking: Nyx v1.0 released
```

Multiple subscribers receive the same message. Subscribers can listen to multiple channels. The connection enters a special subscriber mode where only `SUBSCRIBE`, `UNSUBSCRIBE`, and `PING` are accepted.

---

## Benchmarking

```bash
# Standard redis-benchmark
redis-benchmark -h localhost -p 6380 -t set,get -n 100000 -q

# Pipelined
redis-benchmark -h localhost -p 6380 -t set,get -n 100000 -P 16 -q
```
