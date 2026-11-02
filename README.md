# drockb

**Baga driver for [rocksbaga](../rocksbaga)** — a rust-rocksdb-shaped KV API
over the rocksbaga RESP2 wire protocol.

rocksbaga is a durable LSM KV server; drockb is the client library Baga
programs use to talk to it. It does not link the storage engine and does not
need RocksDB installed — only a TCP connection to a running server.

- Binary-safe: keys and values are `bytes`, never NUL-terminated strings.
- Explicit results: every call returns a struct with `ok` / `err`, so a
  missing key is never confused with a failed command.
- No hidden state: one connection per `DrokDB` handle, reuse it across calls.

## Quick start

```baga
import "drockb/drockb.baga"

fn main() -> i64 !Net !IO {
    let db = drock_open("127.0.0.1", 16579)?
    if drock_ok(db) == 0 {
        eprintln(drock_last_error(db))
        return 1
    }
    drock_put_str(db, "name", "Baga")?
    print(drock_get_str(db, "name")?)
    drock_close(db)?
    return 0
}
```

Start the server (from the baga monorepo root):

```bash
LSMPATH=/tmp/rocksdb_data LSMPORT=16579 \
  ./baga -I . -I app-product app-product/rocksbaga/tools/serve.baga
```

Run the bundled demo:

```bash
./baga -I . -I app-product app-product/drockb/examples/demo.baga
# env: DROCK_HOST (127.0.0.1), DROCK_PORT (16579)
```

## API

### Connection

| function | notes |
|---|---|
| `drock_open(host, port)` | default options |
| `drock_open_with(opts, host, port)` | timeouts, `SELECT` index, scan count |
| `drock_options_default()` + `drock_options_timeout` / `_connect_timeout` / `_db` / `_auto_reconnect` / `_scan_count` | immutable option builders |
| `drock_ping`, `drock_select`, `drock_reconnect` | liveness / DB index / reconnect |
| `drock_ok`, `drock_closed`, `drock_last_error`, `drock_close` | handle state |

### Keys and values

`drock_put`, `drock_put_str`, `drock_get` → `DrokValue{ok,found,val}`,
`drock_get_str`, `drock_delete`, `drock_del` (multi-key),
`drock_exists`, `drock_getset`, `drock_getdel`, `drock_setnx`.

Counters and strings: `drock_incr`, `drock_decr`, `drock_incrby`,
`drock_decrby`, `drock_append`, `drock_strlen`.

TTL: `drock_put_ex` (SETEX), `drock_expire`, `drock_ttl`, `drock_persist`.

Admin: `drock_type`, `drock_dbsize`, `drock_save`, `drock_bgsave`,
`drock_flushdb`, `drock_flushall`, `drock_checkpoint`, `drock_backup`.

Multi-key: `drock_mset`, `drock_mget` → `DrokMulti{found, vals}`,
`drock_keys`.

### Batch

```baga
let b = drock_batch_new()
drock_batch_put(b, bytes_of_str("k1"), bytes_of_str("v1"))
drock_batch_delete(b, bytes_of_str("k2"))
drock_batch_write(db, b)?
```

Operations run in insertion order and stop at the first error. **This is not
a transaction** — rocksbaga has no MULTI/EXEC. See [gaps.md](gaps.md).

### Iterators (SCAN-backed)

```baga
let mut it = drock_iter_prefix(drock_iter_new(db), bytes_of_str("user:"))
it = drock_iter_next(it)?
while drock_iter_valid(it) == 1 {
    let v = drock_iter_value(it)?
    it = drock_iter_next(it)?
}
drock_iter_close(it)
```

`drock_iter_match` uses server-side glob `MATCH`; `drock_iter_prefix` filters
exactly on the client. SCAN is cursor-based and unordered: there are no
snapshots or stable ordering guarantees. The iterator becomes invalid after
the final key.

### Column families (`LSM_CF=1` servers)

`drock_cf_create`, `drock_cf_list`, `drock_cf_put`, `drock_cf_get`,
`drock_cf_del`, `drock_cf_drop`, `drock_cf_setopt`.

CF mode is an extension, not full RocksDB column families: one shared WAL,
and the server exposes only a subset of plain commands. See
[gaps.md](gaps.md) and `tests/cf_test.baga`.

## Tests

```bash
# unit: codec, no server
./baga -I . -I app-product app-product/drockb/tests/wire_test.baga

# integration: default server on :16579
DROCK_PORT=16579 ./baga -I . -I app-product app-product/drockb/tests/smoke_test.baga

# integration: server started with LSM_CF=1
DROCK_PORT=16579 ./baga -I . -I app-product app-product/drockb/tests/cf_test.baga
```

`make test` runs the unit test plus both integration suites against isolated
rocksbaga servers (see `scripts/run_tests.sh`).

## Layout

| file | contents |
|---|---|
| `types.baga` | options and result structs |
| `wire.baga` | binary-safe RESP2 encoder/decoder |
| `conn.baga` | connection state, transport, reply adapters |
| `client.baga` | public KV API |
| `batch.baga` | ordered (non-atomic) write batch |
| `iter.baga` | SCAN-backed iterator |
| `cf.baga` | column-family commands |
| `tests/wire_test.baga` | pure codec checks |
| `tests/smoke_test.baga` | default-server integration |
| `tests/cf_test.baga` | `LSM_CF=1` integration |

Known limitations: [gaps.md](gaps.md).
