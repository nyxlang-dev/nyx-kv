# nyx-kv

Redis-compatible key-value store library for [Nyx](https://nyxlang.com).
Implements 52 RESP2 commands across 4 data types (strings, lists, sets, hashes),
multi-tenant AUTH, Pub/Sub, binary `.ndb` persistence, and optional TLS.
Consume it as a package to embed a full KV engine in your own service.

Librería de almacenamiento clave-valor compatible con Redis, escrita en Nyx.
Soporta 52 comandos RESP2, 4 tipos de datos, AUTH multi-tenant, Pub/Sub,
persistencia binaria `.ndb` y TLS opcional. Se consume como paquete PM.

---

## Install

Install the Nyx toolchain:

```bash
curl -sSf https://nyxlang.com/install.sh | sh
```

## Quick start

```bash
git clone https://github.com/nyxlang-dev/nyx-kv
cd nyx-kv
nyx build
./nyx-kv [flags]
```

## Usage

Build and run the standalone smoke test:

```bash
make build               # compiles examples/standalone.nx
./nyx-kv                 # plain RESP2 on :6380, no rate limit, no TLS
redis-cli -p 6380 PING   # → PONG
```

Connect with any Redis client:

```bash
redis-cli -p 6380
> SET hello world
OK
> GET hello
"world"
> HSET user:1 name Alice age 30
(integer) 2
> HGET user:1 name
"Alice"
> SUBSCRIBE events
(Reading messages...)
```

Try the managed free tier:

```bash
redis-cli -h nyxkv.com -p 6380 --tls -a $TOKEN
```

Get a token by signing up at [dashboard.nyxkv.com](https://dashboard.nyxkv.com).

Free tier: 1 000 req/s · 10 000 keys · 1 MB max value · 24 h TTL.

### Embed as a library

```toml
# nyx.toml
[dependencies]
nyx-kv = "*"
```

```nyx
import "std/resp"
import "nyx-kv/src/store"
import "nyx-kv/src/limits"
import "nyx-kv/src/auth"
import "nyx-kv/src/commands"
import "nyx-kv/src/persist"
import "nyx-kv/src/pubsub"

fn main() {
    init_limits()
    // tcp_listen / worker threads / accept loop calling dispatch_command
}
```

## Configuration

| Flag | Default | Description |
|------|---------|-------------|
| `--port N` | `6380` | TCP port to listen on |
| `--public` | off | Enable free-tier rate limiting (1 000 req/s, 10 000 keys, 1 MB values, 24 h TTL) |
| `--tls` | off | Enable TLS — reads cert/key from `TLS_CERT` / `TLS_KEY` env vars |

```bash
./nyx-kv --port 6390
./nyx-kv --public
./nyx-kv --public --tls
TLS_CERT=/path/to/cert.pem TLS_KEY=/path/to/key.pem ./nyx-kv --tls
```

## Performance

```
SET:      82 000 req/s    p50: 0.30 ms
GET:      85 000 req/s    p50: 0.30 ms
INCR:     71 000 req/s    p50: 0.34 ms
SADD:     59 000 req/s    p50: 0.39 ms
HSET:     61 000 req/s    p50: 0.38 ms
SET P=50: 253 000 req/s   (pipelined)
```

Measured on AWS `t4g.micro` ARM64, single worker.

## Documentation

See [docs/](./docs/) for full reference:

- [docs/ARCHITECTURE.md](./docs/ARCHITECTURE.md) — storage engine internals, module map
- [docs/COMMANDS.md](./docs/COMMANDS.md) — all 52 RESP2 commands with signatures

## Limitations

- `LPUSH` under high concurrency uses reversed internal storage as a workaround for a `memmove` race
- Nested map values cause SEGV — use flat key encoding (`"key::field"`) instead
- `PERSIST` command not implemented
- Free tier has no per-IP connection limit
- Pro/Enterprise tiers are not enforced — `--public` flag is binary, no billing or plan enforcement

## License

Apache 2.0 — see [LICENSE](../../LICENSE)
