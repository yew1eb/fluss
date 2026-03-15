# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Apache Fluss (Incubating) is a streaming storage system for real-time analytics built on Java 11. It bridges data streaming and data lakehouse architectures, integrating with Apache Flink (primary), Apache Spark, and lake formats (Paimon, Iceberg, Lance).

Current version: `0.10-SNAPSHOT`

## Build Commands

```bash
# Full build, skip tests
./mvnw clean install -DskipTests

# Parallel build (faster)
./mvnw clean install -DskipTests -T 1C

# Run all tests for a specific module
./mvnw verify -pl fluss-server

# Run unit tests only (pattern: **/*Test.*)
./mvnw test -pl fluss-server

# Run integration tests only (pattern: **/*ITCase.*)
./mvnw verify -pl fluss-server -Dtest=None -Dit.test="**/*ITCase*"

# Run a single test class
./mvnw test -pl fluss-server -Dtest=TabletServerTest

# Apply code formatting
./mvnw spotless:apply

# Run checkstyle
./mvnw checkstyle:check
```

## Code Style & Quality

- **Formatter**: Spotless + google-java-format v1.7.0.6 (AOSP style). Run `./mvnw spotless:apply` before committing.
- **Checkstyle**: Config at `tools/maven/checkstyle.xml` with suppressions at `tools/maven/suppressions.xml`. Version 9.3.
- **License headers**: All Java/Scala files must have the Apache License 2.0 header. Checked by apache-rat plugin at `validate` phase.
- **Test naming**: Unit tests match `**/*Test.*`; integration tests match everything else (typically `**/*ITCase.*`).

## Architecture

### Cluster Components

A Fluss cluster has two server processes:

- **CoordinatorServer** (`fluss-server/.../server/coordinator/`): The cluster brain. Manages metadata, tablet allocation, rebalancing, table DDL, and failure recovery via ZooKeeper. Uses a state machine model for bucket/replica leadership.
- **TabletServer** (`fluss-server/.../server/tablet/`): Data plane. Handles storage and I/O for both LogStore and KvStore. Each table bucket is a **tablet** containing a LogTablet (always) and optionally a KvTablet (PrimaryKey tables only).

### Storage Layer (fluss-server)

- **LogStore** (`server/log/`): Append-only log, similar to Kafka partitions. Supports log replication, remote tiering.
- **KvStore** (`server/kv/`): RocksDB-backed key-value store for PrimaryKey tables. Supports partial updates, row mergers, WAL-based recovery from LogStore, and periodic snapshots to remote storage.
- **Replica** (`server/replica/`): Manages leader/follower replication for LogTablets. KvTablets currently have no replication.
- **ZooKeeper** (`server/zk/`): Used for cluster coordination and metadata storage (planned to be replaced by Raft + KvStore).

### Table Types

- **Log Table**: Append-only. Only LogStore activated. Optimized for write-heavy streaming.
- **PrimaryKey Table**: Supports upserts. Both LogStore (changelog) and KvStore (current state) activated.

### Module Structure

| Module | Purpose |
|--------|---------|
| `fluss-common` | Shared types, config, row formats (Arrow/columnar/indexed), RPC messages, metadata, filesystem abstractions, security |
| `fluss-rpc` | Netty-based RPC framework, gateway interfaces, request/response protocol |
| `fluss-client` | Java client SDK (admin, table writer, log scanner, kv lookup) |
| `fluss-server` | CoordinatorServer + TabletServer implementations |
| `fluss-flink/fluss-flink-common` | Flink connector (source, sink, catalog, tiering, lookup) — targets latest Flink |
| `fluss-flink/fluss-flink-{1.18,1.19,1.20,2.2}` | Version-specific Flink compatibility shims |
| `fluss-flink/fluss-flink-tiering` | Standalone Flink job entrypoint for lake tiering (separate jar to avoid classloader conflicts) |
| `fluss-spark/fluss-spark-common` | Spark connector common logic |
| `fluss-spark/fluss-spark-{3.4,3.5}` | Spark version-specific modules |
| `fluss-lake/fluss-lake-{paimon,iceberg,lance}` | Lake format integrations for tiered storage |
| `fluss-filesystems` | Pluggable filesystem implementations (S3, OSS, HDFS, Azure, GCS, OBS) |
| `fluss-metrics` | Metrics reporters (JMX, Prometheus) |
| `fluss-protogen` | Custom protobuf-like code generator used for RPC message serialization |
| `fluss-kafka` | Kafka protocol compatibility layer |
| `fluss-test-utils` | Shared test infrastructure (embedded cluster, test bases) |
| `fluss-jmh` | JMH benchmarks |

### Multi-Version Flink Support

`fluss-flink-common` contains all connector logic targeting the latest Flink. Version-specific modules (`fluss-flink-1.18` etc.) add compatibility shims for API changes between Flink versions. When Flink APIs change, compatibility code goes in the version-specific module, not in `fluss-flink-common`.

### Tiering Service

Fluss data can be tiered to lake formats (Paimon, Iceberg) via a Flink streaming job. The flow: `fluss-flink-tiering` (job entrypoint) → `fluss-flink-common` tiering source/committer → `fluss-lake-*` writers. This is a background Flink job, not part of the main server.

### RPC Layer

`fluss-protogen` generates custom serialization code from `.proto`-style definitions. `fluss-rpc` provides Netty-based transport with gateway interfaces defined in `rpc/gateway/`. Server-side handlers live in `fluss-server`; client-side in `fluss-client`.

## PR Conventions

- PR title format: `[component] Title` (e.g., `[kv]`, `[log]`, `[client]`, `[flink]`, `[lake/paimon]`)
- Hotfixes: `[hotfix][component] Title`
- Each PR must correspond to a GitHub issue (except doc/JavaDoc typos)
- Run `./mvnw clean verify` to validate before opening a PR

## CI Test Stages

CI splits tests into four parallel stages:
- **core**: `fluss-common`, `fluss-client`, `fluss-server`, `fluss-rpc`, etc.
- **flink**: `fluss-flink-common`, `fluss-flink-2.2`, `fluss-flink-1.20`
- **spark3**: `fluss-spark-*`
- **lake**: `fluss-flink-1.18`, `fluss-flink-1.19`, `fluss-lake-*`
