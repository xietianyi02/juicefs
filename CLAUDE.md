# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

JuiceFS is a high-performance POSIX file system for cloud-native environments. It splits file data into chunks stored in object storage (S3, etc.) while persisting metadata in engines like Redis, MySQL, TiKV, or SQLite. Written in Go.

## Build Commands

```bash
make                    # Build the juicefs binary (release mode, stripped)
make debug              # Build with debug symbols (no optimization)
make juicefs.lite       # Build with most backends disabled (minimal binary)
make juicefs.ceph       # Build with Ceph support (requires ceph build tag)
make juicefs.fdb        # Build with FoundationDB support
```

## Testing

```bash
# Unit tests by package area
make test.meta.core             # Core metadata tests (skips non-core)
make test.meta.non-core         # Redis cluster, PostgreSQL, etcd, KeyDB tests
make test.pkg                   # All pkg/ tests except meta (includes gluster tag)
make test.cmd                   # Command-level integration tests (requires sudo, minio env vars)
make test.fdb                   # FoundationDB-specific meta tests

# Run a single test
go test -v -run TestName -count=1 ./pkg/meta/...
go test -v -run TestName -count=1 ./cmd/...

# Random/fuzz-style testing
make unit-random-test meta=redis seed=12345 checks=100 steps=1000
```

Note: `test.cmd` requires `sudo` and environment variables `MINIO_ACCESS_KEY=testUser MINIO_SECRET_KEY=testUserPassword`.

## Linting

Uses golangci-lint via pre-commit hooks. Run `pre-commit install` to set up. Format with `go fmt`.

The golangci config (`.golangci.yml`) disables `govet` and suppresses several staticcheck quickfix rules (QF1001, QF1003, QF1008, QF1011, QF1012) and ST1005 (error string capitalization).

## Architecture

### Data Flow: Client → VFS → Chunk → Object Storage

- **`cmd/`** — CLI commands using `urfave/cli/v2`. Each subcommand (format, mount, gc, fsck, sync, etc.) is a separate file. `cmd/main.go` registers all commands.
- **`pkg/vfs/`** — Virtual file system layer implementing POSIX semantics. Coordinates between metadata and chunk storage. Handles read/write buffering, file handles, compaction.
- **`pkg/meta/`** — Metadata engine abstraction. `interface.go` defines the `Meta` interface. Implementations:
  - `redis.go` — Redis/KeyDB (build tag `!noredis`)
  - `sql.go` — SQL-based engines with per-DB files: `sql_mysql.go`, `sql_pg.go`, `sql_sqlite.go`
  - `tkv_*.go` — Key-value stores: TiKV, BadgerDB, etcd, FoundationDB (fdb requires opt-in `fdb` build tag)
- **`pkg/chunk/`** — Data chunk management. Handles splitting files into 64MB chunks → 4MB blocks, disk caching, memory caching, and eviction.
- **`pkg/object/`** — Object storage abstraction. `interface.go` defines the `ObjectStorage` interface. One file per backend (S3, Azure, GCS, OSS, HDFS, Ceph, local file, etc.). Backends are excluded via `no<name>` build tags.
- **`pkg/fuse/`** — FUSE mount implementation.
- **`pkg/fs/`** — High-level filesystem operations and HTTP interface.
- **`pkg/sync/`** — Data sync between storage systems.
- **`pkg/gateway/`** — S3-compatible gateway.
- **`sdk/java/`, `sdk/python/`** — Language SDKs (JNI/Python bindings).

### Build Tags

Build tags control which backends are compiled in. Most use a negative pattern (`noredis`, `nosqlite`, `nomysql`, etc.) — backends are included by default and excluded with `no*` tags. Exceptions: `ceph`, `fdb`, `gluster` require explicit opt-in tags.

### Key Interfaces

- **`meta.Meta`** (`pkg/meta/interface.go`) — All metadata operations (file create/read/write, directory ops, locking, xattrs, ACLs).
- **`object.ObjectStorage`** (`pkg/object/interface.go`) — Object storage CRUD (Get, Put, Delete, List, multipart upload).
- **`chunk.ChunkStore`** (`pkg/chunk/chunk.go`) — Chunk read/write with caching.

## Coding Conventions

- Every new source file must begin with the Apache 2.0 license header.
- Follow [Effective Go](https://go.dev/doc/effective_go) and [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments).
- Logging uses `pkg/utils` logger (logrus-based): `var logger = utils.GetLogger("packagename")`.
