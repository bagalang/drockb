# drockb — known gaps

Wire-level limitations of the current driver, and what would close them.
The server-side engine has its own list: `../rocksbaga/docs/gaps.md`.

## D1 — Batches are ordered, not atomic — **open**

`drock_batch_write` sends the operations as independent SET/DEL commands.
If the connection drops halfway, earlier operations stay applied and later
ones never run. rocksbaga has no MULTI/EXEC and no cross-key transaction.

*Close by:* adding a server-side batch/transaction command (single WAL frame
with a commit marker) and switching `batch.baga` to it.

## D2 — No snapshots or stable iteration — **open**

The iterator is SCAN-backed. It is cursor-based, unordered, and may miss or
repeat keys if the data set changes during iteration. Values are fetched with
a separate GET, so a key can disappear between the SCAN page and the GET.

*Close by:* an ordered `ITER` command with server-held cursor and snapshot
semantics. Not a driver-only change.

## D3 — CF mode is a subset — **open**

With `LSM_CF=1` the server supports `PING`, `QUIT`, `SET`, `GET`, `DEL`,
`INCRBY`, `DECRBY`, `GETSET`, `STRLEN`, `GETDEL`, `SETNX`, `UNLINK`,
`SAVE`, `BGSAVE` and the `CF.*` commands. `EXISTS`, `MSET`, `MGET`, `TYPE`,
`DBSIZE`, `SCAN`, TTL commands and `KEYS` are not available in that mode, so
`iter.baga` cannot be used against a CF server.

*Close by:* porting the missing handlers to `server_cf.baga` (they exist in
`server.baga`).

## D4 — One connection, no pool, no pipelining — **open**

Each `DrokDB` owns a single TCP connection. Commands are request/response,
one at a time; the driver never pipelines. Auto-reconnect is opt-in
(`drock_options_auto_reconnect`) and does not retry the failed command.

*Close by:* a pool in the driver, plus an opt-in `retry_on_reconnect` that
only retries idempotent reads.

## D5 — No pub/sub, blocking ops, or scripts — **open**

rocksbaga does not implement these RESP families. drockb exposes none of
them. `BLPOP`-style blocking reads are out of scope until the server has
them.

## D6 — No shard routing — **open**

`LSM_SHARDS` partitions keys inside the server; the driver always talks to
the one endpoint it was given. There is no client-side sharding or cluster
topology.

## D7 — TTL is server-side only — **open**

`drock_put_ex`, `drock_expire`, `drock_persist` send commands and return the
server's answer. The driver keeps no local expiry clock; timeouts are
whatever the server reports.

## D8 — No compression/encryption negotiation — **open**

Values cross the wire as-is. If transport security is needed, put the
connection behind a TLS tunnel; drockb has no TLS option yet.
