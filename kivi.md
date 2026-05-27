# Kivi — a persistent key-value store in Go

Kivi is a from-scratch, disk-backed key-value store written in Go. The goal is to build something that actually works the way production storage engines work — not a toy wrapper around a map, but a real system that survives crashes, handles concurrent readers and writers correctly, persists data to disk in a format that can be recovered, and eventually scales across multiple nodes.

The project is broken into phases. Each phase produces a fully working, testable system before the next one begins. Nothing is scaffolding — every piece of code written in phase N is real code that phase N+1 builds on.

---

## What we are building

### In-memory store

The starting point is a simple in-memory key-value store backed by a Go map and protected by a `sync.RWMutex`. This is the foundation everything else sits on. It handles concurrent readers without blocking each other, serialises writers, and gives us a clean interface — `Get`, `Set`, `Delete`, `Scan` — that the rest of the system will speak.

We also implement TTL (time-to-live) expiry here. Each key can be given an expiry time. A background goroutine periodically reaps expired keys. This teaches the pattern of background workers running alongside the main request path, which comes up constantly in storage systems.

### Append-only log

Before we can survive a restart, we need to persist writes to disk. The simplest durable structure is an append-only log: every write is encoded as a binary record and appended to a file. On startup, we read the log front-to-back and replay every record to reconstruct the in-memory state.

The binary encoding is custom — a fixed header with key length, value length, record type, and a CRC32 checksum. The checksum matters: disks lie, filesystems lie, and without a checksum we have no way to detect a corrupted or partially-written record. A fuzz test will try to feed the decoder garbage and verify it never panics and always returns a clean error.

We also expose sync modes. `O_SYNC` vs `fdatasync` vs `fsync` have meaningfully different performance and durability guarantees. A benchmark here makes the trade-off concrete rather than theoretical.

### Write-ahead log (WAL)

The append-only log gives us durability after a clean shutdown. What it does not give us is crash safety during a write. If the process dies mid-write, we might have half a record on disk. The write-ahead log solves this.

The WAL is a separate log written before any mutation reaches the in-memory store. On startup, Kivi replays the WAL to recover any writes that made it to disk but not yet to the main data file. WAL records have types: `put`, `delete`, `checkpoint`. A checkpoint marks the point up to which the WAL has been fully applied and can be discarded.

A crash-test harness kills the process at random points during a write workload and verifies on restart that no committed write was lost and no uncommitted write was applied.

### Bitcask engine — segments and indexes

An append-only log that grows forever is not a storage engine. The Bitcask model splits the log into fixed-size segment files. Only the newest segment is active (accepting writes). Older segments are immutable. Each segment has a corresponding hint file — a compact on-disk index mapping every live key in that segment to the exact byte offset of its most recent value. On startup, Kivi loads the hint files instead of replaying every record, making startup fast regardless of total data size.

Deleted and overwritten keys accumulate dead space in old segments. A background compaction process merges old segments together, keeping only the most recent live value for each key. This reclaims disk space and keeps the number of segment files bounded.

The `ReadAt` vs `io.ReadFull` question is addressed here with benchmarks: `pread` (via `ReadAt`) is safer for concurrent random access because it doesn't move the file's seek position, while `io.ReadFull` is simpler but requires holding a lock across the read.

### LSM Tree engine — MemTable and SSTables

The Bitcask model keeps the entire key index in memory. That works well when the key set is small, but it becomes a problem when the number of keys exceeds available RAM. The Log-Structured Merge Tree (LSM) solves this at the cost of more complex reads.

In an LSM engine, writes go first into a **MemTable** — a sorted, in-memory data structure (a skip list or red-black tree). When the MemTable reaches a configured size threshold, it is flushed to disk as an immutable **SSTable** (Sorted String Table). An SSTable is a file containing key-value pairs in sorted order, plus an index block and a footer. Because keys are sorted, a binary search on the index finds any key in O(log n) disk reads.

Reads are more expensive than in Bitcask: to find a key, Kivi must check the MemTable first, then search SSTables from newest to oldest. **Bloom filters** reduce this cost significantly — each SSTable has a bloom filter that probabilistically tells us whether a key is absent, letting us skip the disk read entirely for most missing-key lookups.

Over time, SSTables accumulate. **Leveled compaction** organises SSTables into levels (L0, L1, L2, ...). L0 files are freshly flushed and can have overlapping key ranges. Compaction picks files from L0 and merges them into L1, producing non-overlapping sorted files. This bounds read amplification (the number of files that need to be checked for any single key) while keeping write amplification manageable.

The LSM engine also has its own WAL for the same reason as the Bitcask engine: the MemTable lives in memory and would be lost in a crash without it.

At the end of this phase, Kivi abstracts both engines behind a `StorageEngine` interface. The engine is selected at startup via configuration. A benchmark comparing both engines under the same read/write workloads makes the trade-off concrete.

### API layer

With a working storage engine, Kivi needs a network interface. Two protocols are implemented: raw HTTP and gRPC.

The HTTP server is written from scratch (no framework) and exposes `GET /key/{k}`, `PUT /key/{k}`, `DELETE /key/{k}`, and `GET /keys?prefix=p`. This forces an understanding of TCP connection management, HTTP/1.1 framing, and connection pooling.

The gRPC interface is defined in a `.proto` file. Codegen produces the Go stubs. Streaming RPCs are used for range scans and key-watch operations — pushing multiple results over a single connection rather than making one request per key.

Request pipelining allows a client to send multiple writes in a single TCP write, reducing round-trips. A benchmark shows the latency reduction over per-request mode.

Graceful shutdown drains in-flight requests, fsyncs the active segment, and only then exits. A `/health` endpoint lets load balancers and orchestrators check readiness.

### Transactions and MVCC

Storage without atomicity is incomplete. Kivi implements multi-key transactions incrementally.

**Read-your-writes sessions** are the starting point: a session object buffers writes locally and flushes them atomically to the WAL as a single batch. This ensures a write is either fully applied or not applied at all.

**Compare-and-swap (CAS)** implements optimistic concurrency: a write succeeds only if the current value of a key matches an expected value. This is the primitive that many higher-level coordination patterns are built on.

**MVCC (Multi-Version Concurrency Control)** assigns a monotonically increasing version number to every write. A read at snapshot version N sees only writes with version ≤ N, making snapshot reads consistent regardless of concurrent writers. Old versions are garbage-collected in the background.

**Serialisable transactions** use two-phase locking across multiple keys. A deadlock detector runs a cycle-detection algorithm on the wait-for graph and aborts one of the conflicting transactions.

### Observability

A production storage system is only as good as its ability to tell you what it is doing.

Structured logging uses Go's `slog` package with JSON output. Trace IDs are generated at request entry and propagated through the entire call chain, so every log line for a single request shares a trace ID.

Prometheus metrics expose latency histograms (p50, p95, p99), operations per second counters, current segment count, and compaction lag. A Grafana dashboard can be pointed at the metrics endpoint immediately.

`pprof` endpoints expose CPU and memory profiles on demand. These are used during development to find hot spots and goroutine leaks — not just added as an afterthought.

The benchmark suite runs `go test -bench` scenarios: random reads, sequential writes, mixed read/write, and prefix scans — run against both the Bitcask and LSM engines so the numbers are directly comparable.

### Replication — Raft

A single-node store is a single point of failure. Kivi implements the Raft consensus algorithm to replicate writes across a cluster of nodes.

Raft is implemented from the paper. Leader election uses randomised timeouts to avoid split votes. AppendEntries RPCs replicate log entries to followers. A write is considered committed once a majority of nodes have persisted it. Followers apply committed entries to their local storage engine.

Single-server membership changes use joint consensus to safely add and remove nodes without risking a split-brain. Learner nodes can join the cluster and replicate the log without participating in elections until they have caught up.

When a follower has fallen so far behind that sending individual log entries would be prohibitively slow, the leader transfers a snapshot of the storage engine directly. This is the `InstallSnapshot` RPC.

### Sharding — horizontal scale

Replication makes Kivi fault-tolerant but every node still holds all the data. Sharding splits the keyspace across multiple Raft groups, each responsible for a contiguous or hash-based range.

A consistent hashing ring assigns keys to shards using virtual nodes. When a shard is added or removed, only a fraction of keys need to move. A proxy/router layer sits in front of the cluster and routes individual requests to the correct shard. Range queries that span shard boundaries are handled with scatter-gather.

Rebalancing moves SSTable or segment files between shards in the background. A two-phase handoff ensures the old shard continues serving reads until the new shard confirms it has fully ingested the data.

Cross-shard transactions use two-phase commit (2PC) with Raft groups as the transaction participants. Coordinator crash recovery handles the case where the coordinator fails between the prepare and commit phases.

### Performance at scale

With a working, distributed, replicated system, the final phase focuses on raw performance.

A block cache (LRU or ARC policy) keeps recently read SSTable blocks in memory, reducing disk I/O for hot keys. The cache hit rate is exposed as a metric.

Block compression (Snappy or zstd) reduces the size of SSTable data blocks on disk and the I/O bandwidth needed to read them, at the cost of CPU. A benchmark measures this trade-off at different compression levels.

`io_uring` replaces the traditional one-syscall-per-operation I/O model with a submission ring that batches multiple I/O operations, reducing context switches and kernel crossings. The throughput delta is benchmarked.

The WAL write path is made lock-free using an MPMC (multi-producer, multi-consumer) ring buffer. Writers atomically claim a slot in the ring, write their record, and signal the flusher without holding a mutex. The contention reduction is measured under high concurrency.

---

## Notes

**Why Go?** Go's goroutine model maps naturally onto the concurrency patterns in a storage engine: background compaction, WAL flusher, reaper goroutine, Raft tick loop. The standard library has everything needed without pulling in heavy dependencies. The race detector (`-race`) catches data races during development before they become bugs in production.

**Why two storage engines?** Bitcask and LSM Tree represent fundamentally different trade-offs. Bitcask keeps the full key index in RAM and pays a flat cost per random read (one disk seek). LSM Tree has no RAM requirement for the key index but pays more for reads due to multi-level SSTable lookups. Building both and benchmarking them under the same workloads produces a concrete, experience-backed answer to the question "which one should I use and when?" — which is exactly the kind of depth that matters.

**Why Raft from scratch?** Using an existing Raft library (like `etcd/raft`) is the right call in production. Building it from scratch is the right call for learning. The implementation here prioritises correctness and understandability over performance optimisations. Once it passes the crash tests and the benchmark suite, it is good enough.

**The `StorageEngine` interface** is the most important design decision in the project. It means every layer above the engine (WAL, transactions, Raft log, snapshot transfer) is written against an interface, not a concrete type. Switching engines is a one-line config change. Testing is easy because a mock engine can be substituted.

**Compaction is the hardest part.** Leveled compaction in the LSM engine involves concurrent reads, concurrent writes, file deletions, and atomic renames. Getting the locking right, handling the case where a compaction crashes halfway through, and ensuring reads never see a gap in the key space — this is where most of the subtle bugs live.

**CRC32 everywhere.** Every record written to disk — WAL records, SSTable data blocks, hint file entries — includes a CRC32 checksum. A record whose checksum does not match is treated as corrupted and either skipped (during replay) or returned as an error (during a live read). This is not optional.

---

## Checklist

### Phase 1 — in-memory store

- [X] Initialise Go module (`go mod init github.com/bitswright/kivi`)
- [X] Set up directory layout: `cmd/kivi/`, `internal/store/`, `internal/wal/`, `internal/segment/`, `internal/lsm/`, `internal/engine/`, `internal/server/`
- [X] Write `Makefile` with targets: `build`, `test`, `bench`, `lint`, `race`
- [ ] Implement `MemStore`
  - [X] Internal `map[string]entry` where `entry` holds value and optional expiry
  - [X] `sync.RWMutex` protecting all map access
  - [X] `Get(key string) ([]byte, bool)`
  - [X] `Set(key string, value []byte)`
  - [X] `Delete(key string)`
  - [ ] `Scan(prefix string) iter.Seq2[string, []byte]` (sorted)
- [ ] Implement TTL
  - [X] `SetWithTTL(key string, value []byte, ttl time.Duration)`
  - [X] Lazy expiry check inside `Get`
  - [ ] Background reaper goroutine with configurable tick interval
  - [ ] `Stop()` method that shuts down the reaper cleanly
- [ ] Write unit tests covering: concurrent reads, concurrent writes, TTL expiry, prefix scan ordering
- [ ] Write benchmarks: `BenchmarkGet`, `BenchmarkSet`, `BenchmarkConcurrentReads`

### Phase 2 — append-only log

- [ ] Define binary record format
  - [ ] Header: record type (1 byte), key length (4 bytes), value length (4 bytes), CRC32 (4 bytes)
  - [ ] Body: key bytes, value bytes
  - [ ] Record types: `RecordPut`, `RecordDelete`, `RecordCheckpoint`
- [ ] Implement encoder: `EncodeRecord(w io.Writer, rec Record) error`
- [ ] Implement decoder: `DecodeRecord(r io.Reader) (Record, error)` with CRC32 validation
- [ ] Fuzz test: feed random bytes to decoder, assert no panic and always returns typed error
- [ ] Implement `LogStore`
  - [ ] Open or create log file with `O_APPEND | O_CREATE | O_WRONLY`
  - [ ] `Append(rec Record) error`
  - [ ] `Replay(fn func(Record) error) error` — reads file front-to-back
  - [ ] `Sync() error` — calls fsync on the file descriptor
- [ ] Implement `SyncMode` config: `SyncNone`, `SyncData` (fdatasync), `SyncFull` (fsync)
- [ ] Reconstruct `MemStore` from log on `Open`
- [ ] Benchmark: write throughput under each sync mode
- [ ] Unit tests: append + replay round-trip, corrupted record skipping, partial write at EOF

### Phase 3 — WAL and crash safety

- [ ] Implement WAL writer
  - [ ] Separate file from data log
  - [ ] `WriteRecord(rec WALRecord) (LSN, error)` — returns log sequence number
  - [ ] `Sync() error`
  - [ ] `Truncate(afterLSN LSN) error` — discard tail after crash point
- [ ] Implement WAL reader / recovery
  - [ ] Iterate records from given LSN
  - [ ] Detect partial record at tail (checksum mismatch or short read) and stop cleanly
  - [ ] Apply put/delete records to MemStore
- [ ] Implement checkpoint
  - [ ] Write checkpoint record to WAL after flushing MemStore to data log
  - [ ] On startup: find last checkpoint LSN, skip WAL records before it, replay remainder
- [ ] WAL rotation
  - [ ] Roll to a new WAL file when current file exceeds size threshold
  - [ ] Delete old WAL files after checkpoint confirms they are applied
- [ ] Crash-test harness
  - [ ] Goroutine that calls `os.Exit(1)` after a random number of writes
  - [ ] Outer test loop: write N keys, crash, reopen, verify all committed keys are present
  - [ ] Verify no uncommitted write is visible after recovery
- [ ] Unit tests: clean shutdown recovery, crash recovery, checkpoint skipping

### Phase 4 — Bitcask engine

- [ ] Implement segment file
  - [ ] Fixed maximum size (configurable, default 64 MB)
  - [ ] Active segment: append writes
  - [ ] Immutable segments: read-only after rotation
  - [ ] Segment rotation: close active, open new active
- [ ] Implement hint file
  - [ ] On segment close: write hint file mapping each live key to `(segmentID, offset, size)`
  - [ ] Binary format with CRC32 per entry
  - [ ] On startup: load hint files to build in-memory index without replaying full log
- [ ] Implement in-memory keydir
  - [ ] `map[string]keyEntry` where `keyEntry = {segmentID, offset, size, expiry}`
  - [ ] Updated on every write; entry removed on delete
- [ ] Implement random read
  - [ ] Look up keydir to get segment ID and offset
  - [ ] Open segment file, `ReadAt(offset, size)` — no seek lock needed
  - [ ] Validate record checksum
- [ ] Benchmark: `ReadAt` vs `io.ReadFull` for concurrent random reads
- [ ] Implement compaction
  - [ ] Background goroutine triggered when dead bytes exceed threshold
  - [ ] Merge immutable segments: for each key, write only the most recent live value to a new segment
  - [ ] Atomically replace old segments with merged segment using `os.Rename`
  - [ ] Update keydir to point to new offsets
  - [ ] Delete old segment and hint files
- [ ] Unit tests: write/read round-trip, deletion, compaction correctness, restart with hint files

### Phase 5 — LSM Tree engine

- [ ] Implement MemTable
  - [ ] Sorted structure: skip list (preferred) or red-black tree
  - [ ] `Put(key, value []byte)`, `Get(key []byte) ([]byte, bool)`, `Delete(key []byte)`
  - [ ] `Iterator() iter.Seq2[[]byte, []byte]` — in sorted key order
  - [ ] Size tracking in bytes; configurable flush threshold
  - [ ] `sync.RWMutex` for concurrent access
- [ ] Implement immutable MemTable
  - [ ] When active MemTable hits threshold, atomically swap it to immutable; open new active MemTable
  - [ ] Immutable MemTable is flushed to SSTable by background goroutine
- [ ] Implement SSTable writer
  - [ ] Data block: sorted key-value pairs with CRC32 per block
  - [ ] Index block: one entry per data block with first key and byte offset
  - [ ] Bloom filter block: serialised bloom filter for all keys
  - [ ] Footer: offsets of index block and bloom filter block
  - [ ] Write to temp file, fsync, rename to final name (atomic)
- [ ] Implement SSTable reader
  - [ ] Load index block and bloom filter on open
  - [ ] `Get(key []byte) ([]byte, bool)`: check bloom filter → binary search index → read data block
  - [ ] `Iterator(start, end []byte) iter.Seq2` for range scan
- [ ] Implement bloom filter
  - [ ] Configurable false-positive rate (default 1%)
  - [ ] Serialise to/from bytes for SSTable embedding
- [ ] Implement WAL for LSM engine (same binary format as Phase 3 WAL)
  - [ ] Write to WAL before mutating MemTable
  - [ ] On startup: replay WAL into fresh MemTable if unflushed MemTable is detected
- [ ] Implement leveled compaction
  - [ ] L0: freshly flushed SSTables; allow up to 4 files before triggering L0→L1 compaction
  - [ ] L1+: non-overlapping key ranges; trigger compaction when level size exceeds threshold
  - [ ] Compaction: merge-sort selected SSTables, write new SSTables, atomically update level manifest
  - [ ] Manifest file: records current level layout (which SSTable files are at which level)
  - [ ] Compaction goroutine runs in background; write path never blocks on compaction
- [ ] Implement `StorageEngine` interface
  - [ ] `Get(key []byte) ([]byte, error)`
  - [ ] `Set(key, value []byte) error`
  - [ ] `Delete(key []byte) error`
  - [ ] `Scan(start, end []byte) iter.Seq2[[]byte, []byte]`
  - [ ] `Sync() error`
  - [ ] `Close() error`
  - [ ] Bitcask engine implements interface
  - [ ] LSM engine implements interface
  - [ ] Engine selected via config at startup
- [ ] Benchmark: same `go test -bench` suite run against both engines; record results in `bench/results.md`
- [ ] Unit tests: MemTable flush, SSTable read/write, bloom filter accuracy, compaction key coverage, WAL recovery

### Phase 6 — HTTP and gRPC API

- [ ] Implement raw HTTP server
  - [ ] TCP listener with `net.Listen`; accept loop with `go handle(conn)` per connection
  - [ ] Request parser: read HTTP/1.1 request line and headers without `net/http`
  - [ ] Routes: `GET /key/{k}`, `PUT /key/{k}`, `DELETE /key/{k}`, `GET /keys?prefix=p`
  - [ ] Connection keep-alive support
  - [ ] `GET /health` returns 200 with engine stats JSON
- [ ] Implement gRPC server
  - [ ] Define `kivi.proto`: `Get`, `Put`, `Delete`, `Scan` (server-streaming), `Watch` (server-streaming)
  - [ ] Run `protoc` codegen; check generated files into `internal/proto/`
  - [ ] Implement server struct satisfying generated interface
  - [ ] `Scan` streams key-value pairs matching a prefix range
  - [ ] `Watch` streams change events (put/delete) for keys matching a prefix
- [ ] Implement request pipelining for HTTP
  - [ ] `PUT /batch` accepts JSON array of put operations, applies atomically to engine
  - [ ] Benchmark: single-key puts vs batched puts at the same total write count
- [ ] Implement graceful shutdown
  - [ ] Catch `SIGTERM` and `SIGINT`
  - [ ] Stop accepting new connections
  - [ ] Wait for in-flight requests to complete (with timeout)
  - [ ] Call `engine.Sync()` and `engine.Close()`
- [ ] Integration tests: HTTP round-trip, gRPC round-trip, pipelined writes, graceful shutdown under load

### Phase 7 — Transactions and MVCC

- [ ] Implement read-your-writes session
  - [ ] Session holds pending write buffer
  - [ ] `session.Set` and `session.Delete` write to buffer
  - [ ] `session.Get` checks buffer first, then engine
  - [ ] `session.Commit` encodes all pending writes as a single WAL batch; applies atomically
  - [ ] `session.Rollback` discards buffer
- [ ] Implement compare-and-swap
  - [ ] `CAS(key, expected, newValue []byte) (bool, error)` — atomically replaces value if current equals expected
  - [ ] Test: concurrent CAS operations on same key; verify exactly one succeeds
- [ ] Implement MVCC
  - [ ] Global monotonic version counter (atomic int64)
  - [ ] Each write tagged with version at commit time
  - [ ] `GetAtVersion(key []byte, version int64) ([]byte, bool)` reads the value as of that version
  - [ ] `Snapshot() int64` returns current version as a snapshot handle
  - [ ] Background GC removes versions older than oldest active snapshot
- [ ] Implement serialisable transactions
  - [ ] Transaction object: read set, write set, snapshot version
  - [ ] On commit: acquire locks on all keys in write set (sorted order to prevent deadlock)
  - [ ] Validate read set: if any key in read set has changed since snapshot, abort transaction
  - [ ] Apply write set; release all locks
  - [ ] Deadlock detector: maintain wait-for graph; run cycle detection; abort youngest transaction in cycle
- [ ] Unit tests: session atomicity, CAS correctness, MVCC snapshot isolation, deadlock detection

### Phase 8 — Observability

- [ ] Implement structured logging
  - [ ] Wrap `slog.Logger` with a trace-ID-aware logger
  - [ ] Generate trace ID (UUID v4) at request entry point
  - [ ] Propagate via `context.Context` through engine calls
  - [ ] JSON output to stdout; configurable log level
- [ ] Implement Prometheus metrics
  - [ ] Register metrics in `internal/metrics/metrics.go`: `RequestDuration`, `RequestsTotal`, `ActiveSegments`, `CompactionLag`, `CacheHitRatio`, `MemTableSize`
  - [ ] Instrument HTTP and gRPC handlers with duration and count
  - [ ] Instrument compaction goroutine with lag gauge
  - [ ] Expose `/metrics` endpoint
- [ ] Implement pprof
  - [ ] Register `/debug/pprof/` routes (CPU, memory, goroutine, block, mutex profiles)
  - [ ] Goroutine leak detector in test suite: record goroutine count at start; assert same count at end
- [ ] Implement benchmark suite
  - [ ] `bench/benchmark_test.go` with reproducible scenarios
  - [ ] `BenchmarkRandomRead`, `BenchmarkSequentialWrite`, `BenchmarkMixedRW`, `BenchmarkPrefixScan`
  - [ ] Run against Bitcask engine and LSM engine; write results to `bench/results.md`
  - [ ] Flame graph from pprof for each benchmark scenario

### Phase 9 — Raft replication

- [ ] Implement Raft core types
  - [ ] `LogEntry`: term, index, command bytes
  - [ ] `State`: follower, candidate, leader
  - [ ] Persistent state: current term, voted-for, log (must survive restart)
  - [ ] Volatile state: commit index, last applied
- [ ] Implement leader election
  - [ ] Randomised election timeout (150–300 ms)
  - [ ] On timeout: increment term, transition to candidate, send `RequestVote` to all peers
  - [ ] Vote granted if term is at least as large and candidate log is at least as up-to-date
  - [ ] On majority votes: transition to leader, send empty `AppendEntries` (heartbeat) immediately
- [ ] Implement log replication
  - [ ] Leader sends `AppendEntries` with entries since follower's `nextIndex`
  - [ ] Follower appends entries if log consistency check passes; returns success and new `matchIndex`
  - [ ] Leader advances `commitIndex` when a majority of `matchIndex` values reach that index
  - [ ] Apply goroutine applies committed entries to storage engine
- [ ] Integrate WAL with Raft log
  - [ ] Raft log entries are written to WAL before sending `AppendEntries`
  - [ ] On restart: replay WAL to reconstruct Raft log; resume from last applied index
- [ ] Implement snapshot
  - [ ] `Snapshot()`: call `engine.Snapshot()` to get serialisable state; write to snapshot file
  - [ ] `InstallSnapshot` RPC: leader streams snapshot file to lagging follower
  - [ ] Follower restores engine state from snapshot; sets `lastIncludedIndex` and `lastIncludedTerm`
- [ ] Implement membership changes
  - [ ] Single-server add: leader writes joint configuration entry, commits, then writes new configuration entry
  - [ ] Single-server remove: same two-phase process
  - [ ] Learner mode: node replicates log but does not vote until it catches up
- [ ] Tests: leader election under network partition, log replication correctness (use deterministic network mock), snapshot install and recovery

### Phase 10 — Sharding

- [ ] Implement consistent hashing
  - [ ] Virtual nodes: each physical node owns K points on the ring (default K=150)
  - [ ] `Assign(key []byte) NodeID` using SHA-256 of key, then find successor on ring
  - [ ] `AddNode(id NodeID)` and `RemoveNode(id NodeID)` rebalance virtual nodes
- [ ] Implement router
  - [ ] Router holds current ring and connection pool to each shard's gRPC endpoint
  - [ ] `Route(key []byte) ShardClient`
  - [ ] `ScatterGather(prefix []byte) iter.Seq2` queries all shards in parallel, merges sorted results
- [ ] Implement rebalancing
  - [ ] When a node is added, identify keys that should move; iterate source shard's scan
  - [ ] Stream key-value pairs to new shard via `MigrateKV` RPC
  - [ ] Two-phase handoff: source serves reads until new shard confirms ingestion complete
  - [ ] Source deletes migrated keys after handoff confirmed
- [ ] Implement cross-shard transactions (2PC)
  - [ ] Coordinator: send `Prepare` to all participant shards; wait for all votes
  - [ ] If all vote yes: send `Commit` to all participants
  - [ ] If any vote no: send `Abort` to all participants
  - [ ] Coordinator crash recovery: coordinator writes intent to WAL before sending `Prepare`; on restart, re-send last phase to all participants
- [ ] Tests: key routing correctness, rebalancing key coverage, 2PC commit and abort paths, coordinator crash recovery

### Phase 11 — Performance

- [ ] Implement block cache
  - [ ] LRU cache keyed by `(segmentID or sstableID, blockOffset)`
  - [ ] Configurable max size in bytes
  - [ ] ARC (Adaptive Replacement Cache) as alternative eviction policy — implement and benchmark against LRU
  - [ ] Expose hit/miss ratio via Prometheus gauge
- [ ] Implement block compression
  - [ ] Add Snappy compression to SSTable data blocks (use `github.com/golang/snappy`)
  - [ ] Add zstd as an alternative (use `github.com/klauspost/compress/zstd`)
  - [ ] Compression level configurable per engine instance
  - [ ] Benchmark: throughput and latency at different compression levels vs uncompressed
- [ ] Implement io_uring (Linux only, build tag `linux`)
  - [ ] Use `github.com/iceber/iouring-go` or syscall wrappers
  - [ ] Replace segment `ReadAt` calls with io_uring submission ring
  - [ ] Benchmark: ops/sec vs standard `pread` under high concurrency
- [ ] Implement lock-free WAL write path
  - [ ] MPMC ring buffer: writers claim slots atomically via CAS on head pointer
  - [ ] Flusher goroutine drains full ring to disk; writers block only if ring is full
  - [ ] Benchmark: WAL write throughput under N concurrent writers vs mutex-based path

### Phase 12 — Production hardening (stretch)

- [ ] TLS and mTLS
  - [ ] Load TLS certificates from configurable paths
  - [ ] Enable mTLS between Raft nodes: verify client certificate against cluster CA
  - [ ] Certificate rotation: SIGHUP triggers reload without dropping existing connections
- [ ] Rate limiting
  - [ ] Token bucket per client IP: configurable rate and burst
  - [ ] When bucket exhausted: return `429 Too Many Requests` immediately (HTTP) or status code (gRPC)
  - [ ] Expose current queue depth as metric; drop requests when depth exceeds threshold (backpressure)
- [ ] Chaos and fault injection
  - [ ] Fault injector middleware: configurable probability of injecting disk errors, delayed writes, dropped RPCs
  - [ ] Network partition simulator: block communication between specified node pairs
  - [ ] Clock skew injection: offset `time.Now()` for a node to test Raft timeout sensitivity
  - [ ] Run full test suite with fault injection enabled; verify all invariants still hold
- [ ] Kubernetes operator
  - [ ] Define `KiviCluster` CRD with fields: `replicas`, `shards`, `storageEngine`, `storageSize`
  - [ ] Operator reconcile loop: create/delete `StatefulSet` pods to match desired replica count
  - [ ] Rolling upgrade: update one pod at a time; wait for Raft majority before proceeding
  - [ ] PVC management: provision persistent volumes per pod; retain on pod deletion
  - [ ] Liveness probe: `/health` endpoint; readiness probe: leader elected and log caught up
