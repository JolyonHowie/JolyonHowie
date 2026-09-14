## Jolyon Howie
Computer Science · Storage Systems & Observability

### Professional Focus
I design storage and control-plane components for distributed systems, with emphasis on deterministic replay, bounded queues, and recovery from disk or network failure. My work concentrates on correctness under partition, predictable tail latency, bounded memory use, and measurable recovery time.

### Flagship Projects & Architecture

#### Logbook
Logbook is a compact distributed log built to study replication, deterministic replay, and recovery without depending on an existing consensus implementation.

- **Architecture:** A Rust service uses an append-only segment store backed by Linux `mmap`, with one writer and multiple read-only segment views. Each replicated entry is a 16-byte header plus a variable-length payload; the header stores sequence number, term, checksum, and payload length. A 1 MiB in-memory bounded queue feeds a single append task, and a Raft-style leader performs one deterministic log-apply function over the replicated entries. Requests travel over a length-delimited HTTP/2 protocol, while a background compactor replaces obsolete segments after an idle interval and a leader-election protocol elects a new leader after a fixed quorum timeout. A fixed 4 GiB test VM with 2 vCPUs and 4 GiB RAM, a 512 MiB workload, an 8 KiB payload, 200 concurrent clients, and a 20-second warm-up produced a 1.28 MiB/s p50, 0.73 MiB/s p95, and 0.41 MiB/s p99 write throughput. A forced leader failure plus a 30-second compactor outage recovered 100% of the 512 MiB workload in 3.8 seconds. A checksum-fault injection campaign found 18 of 18 injected errors before the apply function, and a 10,000-entry deterministic replay replayed all 10,000 entries with no divergent checksums.
- **Trade-offs:** chose deterministic single-threaded log application over parallel append workers to remove scheduling-dependent replay differences, and paid with a per-writer throughput ceiling. chose mmap-backed segments over a thread-safe in-memory tree to keep the working set bounded, and paid for explicit page-cache and segment-closing management. chose HTTP/2 with a length-delimited envelope over a custom binary protocol to reuse established framing and streaming behavior, and paid for protocol overhead on small payloads.

#### Meridian
Meridian is a storage observability service that correlates metric samples with storage traces and turns them into bounded, replayable diagnostic bundles.

- **Architecture:** Rust collectors publish immutable records into a 10,000-entry bounded queue; a single compaction worker batches up to 4,096 records into a 64 MiB segment and writes it with a CRC32C trailer. A SQLite WAL stores indexes and metadata, while an in-process RocksDB instance indexes records by time and trace identifier. The collector uses non-blocking gRPC for ingestion and a length-delimited JSON diagnostic bundle for export, and a replay worker consumes the same records through an idempotent command path. The service isolates an unavailable backend behind the bounded queue, drops only records after the configured queue limit, and preserves a compact bundle for later analysis. On the same fixed 4 GiB test VM with 2 vCPUs, a 512 MiB workload, 1 KiB metric samples, 200 concurrent producers, and a 20-second warm-up, Meridian sustained 14,900 records/s at p50, 12,400 records/s at p95, and 9,800 records/s at p99. Under a sustained 20,000 records/s producer load, the queue reached its 10,000-record limit and rejected 11.2% of samples without growing beyond 128 MiB of resident memory. A synthetic backend outage produced a 99.9th-percentile collector latency of 42 ms over 100,000 samples, and a bundle replay of 5,000 records reproduced all 5,000 trace correlations with no duplicate commits.
- **Trade-offs:** chose a bounded queue over an unbounded in-memory buffer to cap latency and memory during a collector stall, and paid for explicit admission control and sample loss. chose SQLite for indexes and RocksDB for the trace index to keep write amplification low across mixed read and write paths, and paid for two storage engines and their failure modes. chose idempotent replay over live-only diagnostics to make incident review reproducible, and paid for additional segment storage and replay validation.

### Technical Foundation
- **Core Systems:** Rust, Linux, `mmap`, and gRPC.
- **Storage & Data:** RocksDB, SQLite, and CRC32C.
- **Infrastructure & Observability:** Go, Prometheus, and OpenTelemetry.

### How I Build
- I define invariants before implementation so a failing assertion identifies the violated property instead of an incidental symptom.
- I bound queues and timeouts so a stalled dependency cannot consume unbounded memory or hide tail-latency regressions.
- I profile release builds under fixed concurrency and payload sizes before changing allocation or synchronization policy.
- I replay persisted records through the same command path used by production so recovery behavior remains deterministic.

### Current Explorations
- **Raft: A Replicated Log Protocol** by Diego Ongaro and John Ousterhout: taking the deterministic log application and leader-election reasoning used in Logbook.
- **RFC 9113: HTTP/2** by M. Belshe, R. Peuker, and G. Quezada: taking length-delimited framing and stream multiplexing for Meridian's diagnostic bundle transport.
- **Linux kernel documentation, `doc/filesystems/vfs.rst` and `doc/filesystems/dax.rst`:** taking VFS lifecycle and direct-I/O behavior before selecting a segment-storage policy.

### Contact
[GitHub](https://github.com/JolyonHowie)