# lsmkv

A small, crash-safe, embedded key-value store built on a **log-structured merge tree (LSM-tree)**, written in pure Python with no dependencies.

It exists to explore the core design trade-offs behind storage engines like LevelDB and RocksDB in a codebase small enough to read in one sitting (~600 lines).

```python
from lsmkv import DB

with DB("./data") as db:
    db.put(b"user:1", b"ada")
    db.put(b"user:2", b"grace")
    db.delete(b"user:2")
    db.get(b"user:1")            # b"ada"
    db.scan(b"user:", b"user;")  # [(b"user:1", b"ada")]
```

```console
$ python -m lsmkv ./data put hello world
$ python -m lsmkv ./data get hello
world
$ python -m lsmkv ./data scan
hello	world
$ python -m lsmkv ./data stats
```

## Architecture

```
 put/delete ──► WAL (append, crc32) ──► memtable (dict)
                                           │ size >= memtable_bytes
                                           ▼
                                   flush → SSTable (immutable, sorted)
                                           │ count >= compact_threshold
                                           ▼
                                   compaction → single merged SSTable

 get  : memtable → newest SSTable → … → oldest SSTable   (bloom filter skips most misses)
 scan : k-way heap merge across memtable + all SSTables, newest version wins
```

| Module | Responsibility |
|---|---|
| `codec.py` | Length-prefixed record format; tombstones are a sentinel value length |
| `wal.py` | Checksummed write-ahead log; torn-tail detection and repair on recovery |
| `bloom.py` | Bloom filter (double hashing) so lookups skip tables that can't hold a key |
| `sstable.py` | Sorted immutable files: data blocks, sparse index, bloom filter, footer with magic number |
| `db.py` | Memtable, flush, compaction, manifest, process lock, merged scans |
| `cli.py` | `put / get / del / scan / flush / compact / stats` |

## Correctness and crash safety

The interesting part of a storage engine is what happens when it dies at the worst moment.

- **Log before apply.** A write reaches the WAL before the memtable. A process crash never loses an acknowledged write; `sync=True` extends that to power loss by fsyncing each write.
- **Torn writes.** Each WAL entry has a CRC. Recovery stops at the first truncated or corrupt entry and truncates the file there, so later appends stay clean.
- **Manifest as the source of truth.** The set of live SSTables is recorded in a `MANIFEST` file replaced atomically (write temp, fsync, rename, fsync directory). A crash mid-flush or mid-compaction leaves orphan files, which are deleted on next open.
- **Idempotent replay.** If a crash lands between publishing an SSTable and clearing the WAL, replaying the WAL rewrites identical data and is harmless.
- **No tombstone resurrection.** Compaction covers *all* tables before dropping tombstones, and the manifest switch is atomic, so a deleted key can never reappear from an older table after a crash.
- **Process exclusion.** An advisory `flock` on `LOCK` stops two processes opening the same directory (POSIX).

## Trade-offs (deliberate)

- **Full compaction instead of leveled/tiered.** Simple and easy to prove correct, but write amplification grows with data size. Leveled compaction would bound it; the manifest format already supports multiple tables, so it's an extension point, not a rewrite.
- **`scan` returns a list**, not a lazy iterator. It's a consistent snapshot that a concurrent compaction can't invalidate, at the cost of memory for huge ranges (use `limit`).
- **Single global lock** (`RLock`). Correct and simple; reads don't scale across threads. SSTable reads use `pread`, so moving to snapshot-based concurrent reads is feasible.
- **Whole-WAL replay into memory.** Bounded by `memtable_bytes`, so recovery cost is bounded too.
- **Default `sync=False`.** Fast, safe against process crashes; you opt in to fsync-per-write for power-loss durability.
- **Keys and values are `bytes`.** No implicit encoding decisions in the engine.

## Testing

```console
$ python -m unittest discover -s tests -t .
```

The suite includes unit tests per component, crash-recovery scenarios (torn WAL tail, corrupt checksum, orphan files, crash between SSTable publish and WAL reset, tombstone non-resurrection), and a **seeded randomized differential test** that applies thousands of random operations (including reopen and compaction cycles) to both `lsmkv` and a plain `dict`, asserting they always agree.

## Possible next steps

- Leveled compaction with a background thread
- Block-level compression and a block cache
- Snapshot reads / iterators with sequence numbers
- A fault-injection harness that kills the process at every fsync boundary

## License

MIT
