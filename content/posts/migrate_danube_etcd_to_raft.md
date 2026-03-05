---
title: "Replacing ETCD with Embedded Raft in a Distributed Message Broker"
date: 2026-03-04
draft: false
categories: ["Danube Messaging"]
author: "Dan Rusei"

autoCollapseToc: true
contentCopyright: MIT

---

Danube is an open-source distributed messaging platform written in Rust. Until recently, it depended on etcd, an external distributed key-value store, for all cluster metadata: topic assignments, namespace policies, schema registries, broker liveness, and leader election. This article describes why and how we replaced etcd with an embedded Raft consensus layer using [openraft](https://github.com/databendlabs/openraft) and [redb](https://github.com/cberner/redb), eliminating an entire external dependency from the deployment.

## Why Replace etcd?

etcd worked. It gave us linearizable reads and writes, watch-based event streams, lease-based liveness detection, and battle-tested fault tolerance. So why replace it?

**Operational overhead.** Every Danube deployment required a separate etcd cluster, typically 3 or 5 nodes, that had to be provisioned, monitored, and upgraded independently. For a messaging system that already runs its own cluster of brokers, maintaining a second distributed system just for metadata felt heavy.

**Failure detection latency.** etcd uses leases with TTLs for broker liveness. When a broker dies, its lease must expire before the cluster reacts. With a 32-second TTL (configured in the application), failure detection took many seconds. Raft heartbeats can detect failure in 2-3 seconds.

**Deployment simplicity.** With embedded Raft, a Danube cluster is just N broker processes. No etcd to deploy, no separate configuration, no version compatibility matrix. One binary, one cluster, one set of logs.

**The consistency guarantees are identical.** etcd is itself a Raft implementation (over boltdb). Replacing it with another Raft implementation (openraft over redb) provides mathematically the same guarantees, linearizable writes, quorum-based durability, partition tolerance. We're not trading correctness for convenience, we're removing a layer of indirection.

## What Etcd did for Danube

Before diving into the replacement, here's what etcd provided, six distinct usage patterns we needed to replicate:

1. **Broker registration & liveness** — Each broker registered itself with an etcd lease. A background keep-alive task renewed the lease periodically. If the broker died, the lease expired, the key was deleted, and watchers detected the failure.

2. **Leader election** — Brokers competed for a `/cluster/leader` key using lease-based compare-and-swap. The winner became the cluster leader (responsible for topic assignment and rebalancing).

3. **Metadata KV store** — Topics, namespaces, schemas, subscriptions, all stored as JSON values under hierarchical key paths like `/cluster/brokers/{id}/{namespace}/{topic}`.

4. **Watch/event bus** — etcd's prefix watch API powered the broker's event-driven architecture. `LoadManager` and `BrokerWatcher` subscribed to key prefixes and reacted to changes.

5. **Load reporting** — Brokers periodically wrote load metrics; the leader watched for them and made rebalancing decisions.

6. **WAL metadata** — Persistent storage (Write-Ahead Log) metadata was stored in etcd alongside everything else.

## The Replacement: openraft + redb

### Technology Choices

**openraft** is a pure-Rust, async-native Raft implementation used in production by Databend and many others. It provides clean traits, `RaftStateMachine`, `RaftLogStorage`, `RaftNetwork`, that you implement with your own storage and transport. No opinions about how you persist data or send messages.

**redb** is a pure-Rust embedded B-tree database (similar in spirit to SQLite, but for key-value workloads). It provides ACID transactions with fsync, which is exactly what a Raft log store needs. We chose it over RocksDB (C++ FFI, painful cross-compilation).

### Architecture

Every broker now runs a Raft node in-process. There is no external metadata service.

The stack has four layers, top to bottom:

1. **Broker layer** — `danube-broker` calls a `MetadataStore` trait (`get`, `put`, `delete`, `watch`). It doesn't know whether the backend is etcd or Raft.
2. **RaftMetadataStore** — the trait implementation. Reads go to the local state machine (in-memory `BTreeMap`). Writes are proposed through Raft, with automatic leader forwarding. Watch events are delivered via `tokio::sync::broadcast` channels.
3. **openraft::Raft** — the consensus engine. It replicates log entries across peers, manages elections, and drives the state machine (`DanubeStateMachine`: `BTreeMap` + watchers + TTL tracking).
4. **Storage & network** — `log_store` (redb, on-disk) persists the Raft log with ACID transactions. `network.rs` (tonic gRPC) handles peer-to-peer communication for AppendEntries, Vote, and InstallSnapshot RPCs.

The key insight is that **the Raft log and the state machine serve different purposes**:

- The **Raft log** (redb, on-disk) is an append-only sequence of commands. Its job is durability, surviving crashes and enabling replication. Entries are persisted *before* they're committed.

- The **state machine** (BTreeMap, in-memory) is the derived KV store. It's where data becomes "available" for reads. It's updated *after* a quorum has persisted the log entry.

On restart, the persisted log is replayed through the state machine to reconstruct the in-memory state. Periodic snapshots compact the log so restart doesn't require replaying the entire history. .

## Replacing Each etcd Pattern

### Leases → TTL Commands

etcd leases are server-side timers. We replaced them with explicit TTL tracking in the state machine:

- `PutWithTTL { key, value, ttl_ms }` — stores the value with an expiration timestamp
- A background **TTL worker** on the leader periodically scans for expired keys and proposes their deletion through Raft
- The deletion goes through consensus, so all nodes agree on when a key expires
- Watchers see the same `Delete` event they saw with etcd lease expiration

### Leader Election → Free

With etcd, we implemented leader election on top of leases and CAS. With embedded Raft, the Raft leader *is* the cluster leader. This entire subsystem was replaced by a lightweight handle that reads `raft.metrics().current_leader`, no code to maintain.

### Watch Events → Broadcast Channels

etcd's watch API was replaced with `tokio::sync::broadcast` channels. Every time the state machine applies a command, it emits a `WatchEvent` (Put or Delete) to all watchers whose prefix matches the key. The `WatchStream` type is unchanged, downstream consumers (`BrokerWatcher`, `LoadManager`) don't know the difference.

### Metadata KV → Same Interface

The `MetadataStore` trait (`get`, `put`, `delete`, `watch`) remained identical. We simply swapped the implementation from `EtcdStore` to `RaftMetadataStore`. Most files in the `resources/` layer required zero changes.

### Atomic Counters → Raft Command

Schema ID allocation previously used a racy get-then-put pattern. We replaced it with an `AllocateMonotonicId` Raft command that atomically increments and returns a counter within the state machine. No more race conditions under concurrent registration.

## Cluster Formation

One design decision we iterated on was cluster bootstrap. The migration plan originally called for an admin-driven `cluster init` command (similar to `cockroach init`). During implementation, we switched to a simpler **config-driven seed node** approach:

```yaml
meta_store:
  data_dir: "./danube-data/raft"
  seed_nodes:
    - "0.0.0.0:7650"
    - "0.0.0.0:7651"
    - "0.0.0.0:7652"
```

On first boot, each broker generates a random stable `node_id` (persisted to `{data_dir}/node_id`). It then contacts each seed peer via a `GetNodeInfo` gRPC RPC to discover their `(node_id, raft_addr)`. The peer with the lowest `node_id` calls `raft.initialize()`. All nodes wait for leader election to complete before proceeding with broker startup.

For single-node development, omitting `seed_nodes` auto-initializes a single-node cluster with zero configuration. The bootstrap is idempotent, if the cluster is already initialized (persisted membership from a previous run), it returns immediately.

### Dynamic Cluster Membership (`--join`)

Beyond seed-based bootstrap, brokers can join an existing cluster dynamically using the `--join` flag:

```bash
# Start a new broker in join mode
danube-broker --join --config-file broker.yml --raft-addr 0.0.0.0:7652

# From an admin terminal, add it to the cluster
danube-admin cluster add-node --node-addr http://new-broker:50051
danube-admin cluster promote-node --node-id <ID>
danube-admin brokers activate <ID>
```

When `--join` is set, the broker skips cluster bootstrap entirely. Its admin gRPC server starts immediately (so `cluster add-node` can discover the node), while the broker waits for a Raft leader to appear in its metrics, indicating it has been added to the cluster. The broker then registers itself in a **drained** state, meaning it won't receive topic assignments until an administrator explicitly activates it.

This gives operators explicit, staged control over cluster scaling, no automatic topology changes, no surprise rebalancing. The reverse operation (`cluster remove-node`) cleanly removes a broker from Raft membership before shutdown.

## Implementation Challenges

**Serialization.** We initially used bincode for Raft RPC payloads, but `RaftCommand` contains `serde_json::Value` fields which require `deserialize_any`, unsupported by bincode. Switching network serialization to serde_json fixed it. Raft metadata (Vote, LogId) still uses bincode since those are simple fixed-size types.

**Read-after-write on followers.** With etcd, network round-trip latency masked the gap between a write being committed and the local state reflecting it. With embedded Raft, writes return almost instantly but a follower's state machine may lag by milliseconds. This is inherent to replicated systems and negligible in production. Removing `LocalCache` (the old in-process etcd replica) also eliminated an entire class of stale-cache bugs.

**Timeout tuning.** Default openraft timeouts (50ms heartbeat) were too aggressive for gRPC + JSON over a network. We tuned to 500ms heartbeat / 1500-3000ms election timeout, still much faster than etcd lease-based failure detection.

**Write forwarding.** Only the Raft leader can commit writes. We added a `ClientWrite` gRPC RPC so followers transparently proxy writes to the leader. From the broker's perspective, `put()` works on any node.

## Key Advantages for Danube

- **Zero external dependencies** — No etcd cluster to deploy, monitor, or upgrade. A Danube cluster is just N broker processes.
- **Single binary deployment** — One `danube-broker` binary contains the consensus layer, metadata store, and message broker. Build once, deploy anywhere.
- **Faster failure detection** — Raft heartbeats detect broker failure in ~2-3 seconds vs over 20 seconds with etcd lease TTLs.
- **Lower read latency** — Metadata reads hit an in-memory BTreeMap in the same process. No network round-trip, no serialization.
- **Unified cluster membership** — Raft *is* the membership protocol. Explicit roles (voter/learner) and staged join workflow (`--join` → add → promote → activate) give operators full control over scaling.
- **Pure Rust stack** — openraft + redb + tonic + tokio. No CGO, no cross-language FFI, single `cargo build`, single debugging stack.
- **Simpler codebase** — `LocalCache` (an in-process replica of remote etcd data) was eliminated entirely. The Raft state machine *is* the local cache. Watch events use `tokio::sync::broadcast`, no network reconnection logic.

## Learn More

Danube's documentation covers detailed architecture:

- **[Architecture Overview](https://danube-docs.dev-state.com/architecture/architecture/)** — System diagram and component interaction
- **[Scaling the Cluster](https://danube-docs.dev-state.com/concepts/scaling_cluster/)** — Node lifecycle, add/promote/activate workflow

### Get Started

Deploy a 3-broker Raft cluster in seconds with Docker Compose:

```bash
mkdir danube-docker && cd danube-docker
curl -O https://raw.githubusercontent.com/danube-messaging/danube/main/docker/quickstart/docker-compose.yml
curl -O https://raw.githubusercontent.com/danube-messaging/danube/main/docker/danube_broker.yml
docker-compose up -d
```

**[Docker Compose Quickstart →](https://danube-docs.dev-state.com/getting_started/Danube_docker_compose/)** | **[GitHub Repository →](https://github.com/danube-messaging/danube)**

---

If you find Danube Messaging project interesting, consider giving it a ⭐ on **[GitHub](https://github.com/danube-messaging/danube)** , it helps others discover the project and motivates continued development.
