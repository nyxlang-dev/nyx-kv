# nyx-kv — Command Reference

All commands use the [RESP protocol](https://redis.io/docs/reference/protocol-spec/). Connect with `redis-cli -p 6380` or any Redis client library.

---

## Connection & Auth

### PING

Test connection.

```
PING
→ PONG

PING "hello"
→ "hello"
```

**Returns**: Simple string `PONG`, or bulk string if message provided.

---

### AUTH

Authenticate with a token (public mode only). After authentication, all keys are namespaced to the user.

```
AUTH <token>
→ +OK
```

**Returns**: `+OK` on success, error if invalid token.

---

### WHOAMI

Show current user and plan.

```
WHOAMI
→ "myuser (pro)"
```

**Returns**: Bulk string `"user_id (plan)"`, or `"anonymous (free)"` if not authenticated.

---

### TOKEN_CREATE

Create a new auth token (admin only, localhost connections).

```
TOKEN_CREATE <user_id> <plan>
→ "a1b2c3d4e5f6...64 hex chars"

TOKEN_CREATE <user_id> <plan> <ttl_seconds>
→ "a1b2c3d4e5f6...64 hex chars"

TOKEN_CREATE alice pro 86400
→ "..."  (token expires in 24 hours)

TOKEN_CREATE alice pro 0
→ "..."  (token never expires)
```

**Plans**: `free`, `pro`, `enterprise`

**Returns**: 64-character hex token, or error if not localhost or invalid plan.

---

### TOKEN_REVOKE

Revoke an existing auth token immediately (admin only, localhost connections).

```
TOKEN_REVOKE <token>
→ OK

TOKEN_REVOKE nonexistent_token
→ -ERR token not found
```

**Returns**: `+OK` on success, error if token not found or connection is not localhost.

---

### TOKEN_LIST

List all active auth tokens with user info and expiry (admin only, localhost connections).

```
TOKEN_LIST
→ 1) "alice:pro:0"
   2) "bob:free:1743638400"
```

Each entry is `user_id:plan:expires_at`. `expires_at=0` means no expiry; otherwise it's a Unix timestamp.

**Returns**: RESP bulk array of token info strings.

---

## String Commands

### SET

Set a key to a string value.

```
SET key value
→ OK

SET name "Nyx Language"
→ OK
```

**Returns**: `+OK`

---

### GET

Get the value of a key.

```
GET key
→ "value"

GET nonexistent
→ (nil)
```

**Returns**: Bulk string, or `nil` if key doesn't exist.

---

### SETNX

Set key only if it does not already exist.

```
SETNX key value
→ 1  (key was set)
→ 0  (key already existed)
```

**Returns**: Integer `1` if set, `0` if key already exists.

---

### SETEX

Set key with an expiration time (in seconds).

```
SETEX key seconds value
→ OK

SETEX session 3600 "abc123"
→ OK
```

**Returns**: `+OK`

---

### GETSET

Set key to a new value and return the old value.

```
SET counter "10"
GETSET counter "0"
→ "10"
```

**Returns**: Bulk string (old value), or `nil` if key didn't exist.

---

### APPEND

Append a value to a key. Creates the key if it doesn't exist.

```
SET greeting "Hello"
APPEND greeting " World"
→ 11  (new length)
```

**Returns**: Integer (new string length after append).

---

### STRLEN

Get the length of the value stored at a key.

```
SET key "Hello"
STRLEN key
→ 5
```

**Returns**: Integer (string length), `0` if key doesn't exist.

---

### INCR

Increment the integer value of a key by 1.

```
SET counter "10"
INCR counter
→ 11
```

**Returns**: Integer (new value after increment).

---

### DECR

Decrement the integer value of a key by 1.

```
SET counter "10"
DECR counter
→ 9
```

**Returns**: Integer (new value after decrement).

---

### MSET

Set multiple keys to multiple values in a single command.

```
MSET key1 "val1" key2 "val2" key3 "val3"
→ OK
```

**Returns**: `+OK`

---

### MGET

Get the values of multiple keys.

```
MSET a "1" b "2" c "3"
MGET a b c nonexistent
→ 1) "1"
   2) "2"
   3) "3"
   4) (nil)
```

**Returns**: Array of bulk strings (nil for missing keys).

---

## Key Commands

### DEL

Delete one or more keys.

```
DEL key1 key2 key3
→ 2  (number of keys deleted)
```

**Returns**: Integer (number of keys that were removed).

---

### EXISTS

Check if a key exists.

```
EXISTS mykey
→ 1  (exists)
→ 0  (doesn't exist)
```

**Returns**: Integer `1` if exists, `0` if not.

---

### TYPE

Return the type of the value stored at key.

```
SET mystr "hello"
TYPE mystr
→ string

LPUSH mylist "a"
TYPE mylist
→ list

SADD myset "a"
TYPE myset
→ set

HSET myhash f "v"
TYPE myhash
→ hash
```

**Returns**: Simple string: `string`, `list`, `set`, `hash`, or `none`.

---

### EXPIRE

Set a timeout on a key (in seconds). After the timeout, the key is automatically deleted.

```
SET mykey "hello"
EXPIRE mykey 60
→ 1

EXPIRE nonexistent 60
→ 0
```

**Returns**: Integer `1` if timeout was set, `0` if key doesn't exist.

---

### TTL

Get the remaining time to live of a key (in seconds).

```
SET mykey "hello"
EXPIRE mykey 60
TTL mykey
→ 60

TTL nonexistent
→ -2

TTL persistent_key
→ -1
```

**Returns**: Integer — seconds remaining, `-1` if no expiry, `-2` if key doesn't exist.

---

### KEYS

Return all keys matching the pattern (currently only `*` is supported).

```
KEYS *
→ 1) "name"
   2) "counter"
   3) "session"
```

In public mode: authenticated users only see their own keys (prefix stripped), anonymous users only see non-namespaced keys. Limited to 100 results.

**Returns**: Array of bulk strings.

---

## List Commands

### LPUSH

Insert one or more values at the head (left) of the list.

```
LPUSH mylist "world"
→ 1
LPUSH mylist "hello"
→ 2
LRANGE mylist 0 -1
→ 1) "hello"
   2) "world"
```

**Returns**: Integer (length of list after push).

---

### RPUSH

Insert one or more values at the tail (right) of the list.

```
RPUSH mylist "hello"
→ 1
RPUSH mylist "world"
→ 2
```

**Returns**: Integer (length of list after push).

---

### LPOP

Remove and return the first element of the list.

```
RPUSH mylist "a" "b" "c"
LPOP mylist
→ "a"
```

**Returns**: Bulk string, or `nil` if list is empty.

---

### RPOP

Remove and return the last element of the list.

```
RPUSH mylist "a" "b" "c"
RPOP mylist
→ "c"
```

**Returns**: Bulk string, or `nil` if list is empty.

---

### LLEN

Return the length of the list.

```
RPUSH mylist "a" "b" "c"
LLEN mylist
→ 3
```

**Returns**: Integer (list length), `0` if key doesn't exist.

---

### LRANGE

Return elements from a list within the given range (inclusive).

```
RPUSH mylist "a" "b" "c" "d" "e"
LRANGE mylist 0 2
→ 1) "a"
   2) "b"
   3) "c"

LRANGE mylist -2 -1
→ 1) "d"
   2) "e"

LRANGE mylist 0 -1
→ (all elements)
```

**Returns**: Array of bulk strings.

---

### LINDEX

Return the element at index in the list. Negative indices count from the end.

```
RPUSH mylist "a" "b" "c"
LINDEX mylist 0
→ "a"
LINDEX mylist -1
→ "c"
```

**Returns**: Bulk string, or `nil` if index is out of range.

---

## Set Commands

### SADD

Add one or more members to a set.

```
SADD myset "hello"
→ 1
SADD myset "world"
→ 1
SADD myset "hello"
→ 0  (already exists)
```

**Returns**: Integer (number of members actually added).

---

### SREM

Remove one or more members from a set.

```
SADD myset "a" "b" "c"
SREM myset "a" "b"
→ 2
```

**Returns**: Integer (number of members removed).

---

### SISMEMBER

Check if a value is a member of a set.

```
SADD myset "hello"
SISMEMBER myset "hello"
→ 1
SISMEMBER myset "world"
→ 0
```

**Returns**: Integer `1` if member, `0` if not.

---

### SMEMBERS

Return all members of a set.

```
SADD myset "a" "b" "c"
SMEMBERS myset
→ 1) "a"
   2) "b"
   3) "c"
```

**Returns**: Array of bulk strings (unordered).

---

### SCARD

Return the number of members in a set.

```
SADD myset "a" "b" "c"
SCARD myset
→ 3
```

**Returns**: Integer (set cardinality).

---

## Hash Commands

### HSET

Set one or more field-value pairs in a hash.

```
HSET user name "Ottavio" age "30" lang "Nyx"
→ 3  (new fields added)

HSET user age "31"
→ 0  (field updated, not new)
```

**Returns**: Integer (number of new fields added).

---

### HGET

Get the value of a field in a hash.

```
HSET user name "Ottavio"
HGET user name
→ "Ottavio"

HGET user nonexistent
→ (nil)
```

**Returns**: Bulk string, or `nil` if field or hash doesn't exist.

---

### HDEL

Delete one or more fields from a hash.

```
HSET user name "Ottavio" age "30"
HDEL user age
→ 1
```

**Returns**: Integer (number of fields removed).

---

### HGETALL

Return all field-value pairs of a hash.

```
HSET user name "Ottavio" lang "Nyx"
HGETALL user
→ 1) "name"
   2) "Ottavio"
   3) "lang"
   4) "Nyx"
```

**Returns**: Array of alternating field-value bulk strings.

---

### HKEYS

Return all field names in a hash.

```
HSET user name "Ottavio" lang "Nyx"
HKEYS user
→ 1) "name"
   2) "lang"
```

**Returns**: Array of bulk strings.

---

### HVALS

Return all values in a hash.

```
HSET user name "Ottavio" lang "Nyx"
HVALS user
→ 1) "Ottavio"
   2) "Nyx"
```

**Returns**: Array of bulk strings.

---

### HEXISTS

Check if a field exists in a hash.

```
HSET user name "Ottavio"
HEXISTS user name
→ 1
HEXISTS user nonexistent
→ 0
```

**Returns**: Integer `1` if field exists, `0` if not.

---

### HLEN

Return the number of fields in a hash.

```
HSET user name "Ottavio" lang "Nyx"
HLEN user
→ 2
```

**Returns**: Integer (number of fields).

---

## Pub/Sub Commands

### SUBSCRIBE

Subscribe to one or more channels. The connection enters subscriber mode — only `SUBSCRIBE`, `UNSUBSCRIBE`, and `PING` commands are accepted.

```
SUBSCRIBE channel1
→ *3
→ $9
→ subscribe
→ $8
→ channel1
→ :1

# Messages arrive as:
→ *3
→ $7
→ message
→ $8
→ channel1
→ $13
→ Hello, world!
```

**Returns**: RESP array confirmation. Connection blocks waiting for messages.

---

### PUBLISH

Publish a message to a channel. All subscribers receive it.

```
PUBLISH channel1 "Hello, world!"
→ :2
```

**Returns**: Integer — the number of subscribers that received the message.

---

### UNSUBSCRIBE

Unsubscribe from a channel (only valid in subscriber mode).

```
UNSUBSCRIBE channel1
→ *3
→ $11
→ unsubscribe
→ $8
→ channel1
→ :0
```

**Returns**: RESP array confirmation with remaining subscription count.

---

## Server Commands

### DBSIZE

Return the number of keys in the database.

```
DBSIZE
→ 42
```

In public mode: returns count scoped to the user's namespace.

**Returns**: Integer.

---

### FLUSHDB

Delete all keys in the database. Disabled in public mode.

```
FLUSHDB
→ OK
```

**Returns**: `+OK`

---

### INFO

Return server information and statistics.

```
INFO
→ # Server
  nyx_kv_version:0.2.0
  # Stats
  total_commands_processed:12345
  # Keyspace
  db0:keys=42,list_keys=3,set_keys=2,hash_keys=5
```

**Returns**: Bulk string.

---

### SAVE

Synchronous snapshot to `dump.ndb`. Disabled in public mode.

```
SAVE
→ OK
```

**Returns**: `+OK`

---

### BGSAVE

Trigger a background save. Disabled in public mode.

```
BGSAVE
→ Background saving started
```

**Returns**: Simple string.

---

### LASTSAVE

Return the UNIX timestamp of the last successful save.

```
LASTSAVE
→ 1711929600
```

**Returns**: Integer (unix timestamp in seconds).

---

### CONFIG

Returns empty array. Exists for redis-benchmark compatibility. Disabled in public mode.

```
CONFIG GET save
→ (empty array)
```

---

### COMMAND

Returns empty array. Exists for redis-benchmark/redis-cli compatibility.

```
COMMAND
→ (empty array)

COMMAND DOCS
→ (empty array)
```

---

## TLS Mode

nyx-kv supports TLS-encrypted connections. Start with:

```bash
./nyx-kv --tls
```

The server reads certificate paths from environment variables:

| Variable | Default |
|----------|---------|
| `TLS_CERT` | `/etc/letsencrypt/live/nyxkv.com/fullchain.pem` |
| `TLS_KEY` | `/etc/letsencrypt/live/nyxkv.com/privkey.pem` |

**Connect with TLS:**

```bash
redis-cli -h localhost -p 6380 --tls --cacert /path/to/ca.crt

# Python
r = redis.Redis(host='localhost', port=6380, ssl=True, ssl_ca_certs='/path/to/ca.crt')

# Node.js (ioredis)
const r = new Redis({ port: 6380, tls: { ca: fs.readFileSync('/path/to/ca.crt') } })
```

TLS mode is compatible with all commands. `--public` and `--tls` can be combined.

---

## Plan Limits (Public Mode)

| Limit | Free | Pro | Enterprise |
|-------|------|-----|------------|
| Rate limit | 100 req/s | 10,000 req/s | Unlimited |
| Max keys | 1,000 | 100,000 | Unlimited |
| Max value size | 100KB | 1MB | Unlimited |
| TTL enforcement | 72 hours | None | None |
| SAVE/BGSAVE | Disabled | Enabled | Enabled |
| CONFIG | Disabled | Enabled | Enabled |
| FLUSHDB | Disabled | Enabled | Enabled |
