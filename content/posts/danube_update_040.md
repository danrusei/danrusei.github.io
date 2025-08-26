---
title: "Danube Messaging - Release v0.4.0"
date: 2025-08-26
draft: false
categories: ["Danube Messaging"]
author: "Dan Rusei"

autoCollapseToc: true
contentCopyright: MIT

---

[Danube Messaging](https://github.com/danube-messaging/danube) is a Rust-based distributed pub/sub messaging platform inspired by Apache Pulsar. It supports reliable and non-reliable dispatch, ETCD-backed metadata, and pluggable storage.

Latest improvements focus on performance, reliability, and developer experience, including async multi-threaded processing, TLS/JWT security, and enhanced CLI capabilities.

### Concurrent Topic Processing and Routing

- **Async, multi-threaded topic handling**: Introduced async message processing with a `TopicWorkerPool` to fan topics across workers, removing single-threaded bottlenecks and enabling parallelism on hot topics.
- **Single source of truth for topics**: Centralized topic ownership in `TopicWorkerPool` so all publish/subscribe/ack flows route through the same control point, eliminating scattered state and reducing contention.
- **Routing correctness**: Ensured topics are routed to workers before async ops begin to avoid “topic not found in worker” races; aligned consistent hashing across add/has lookups to keep placement deterministic.

Impact:

- Higher throughput from parallel handling.
- Fewer routing races and clearer operational behavior.
- Predictable scaling under load due to deterministic worker placement.

### Reliability, Metadata, and Lifecycle

- **Reliable dispatch: store-then-notify**: Enforced persistence-first on the reliable path, then notify consumers. This removes double-work, and reduces tail latency.
- **Topic lifecycle management**: Finalized deletion flows (remove from pool, gracefully disconnect producers/consumers, purge indices) and cleaned up inactive subscription metadata at topic and broker levels.
- **Selective interior mutability**: Reduced coarse-grained locking by applying interior mutability only where needed (producer/consumer maps), lowering mutex contention on hot paths.

Impact:

- Lower tail latencies and improved durability semantics.
- Cleaner state transitions and faster topic/subscription churn.

### [Admin CLI](https://github.com/danube-messaging/danube/tree/main/danube-admin-cli) revamp

- **TLS support**: Added TLS options for secure connections to brokers with self-signed or CA certs.
- **Topics with reliable dispatch**: Enabled creating topics with reliable dispatch from the CLI.
- **Improved UX**: Added commands for listing brokers, namespaces, topics, and subscriptions. Enhanced help text and error messages.

---

## Other Danube Changes worth mentioning since last update

- **End-to-end security: TLS transport + JWT authentication**
  - Secure broker–client transport and token-based auth with configurable certs and comprehensive test coverage.

- **Persistent, pluggable storage backends (Disk, S3, TiKV)**
  - Durable message storage with multiple backends behind a clean abstraction, improving reliability and deployment flexibility.

- **Admin plane security hardening (TLS)**
  - Secures administrative gRPC operations to protect cluster management traffic.

- **Asynchronous, multi-threaded broker processing + worker-pool routing**
  - Eliminates single-threaded bottlenecks and centralizes topic control, significantly boosting throughput and reducing routing errors.

- **Reliable dispatch revamp**
  - Delivery guarantees via persistence-first semantics and tighter broker/dispatcher integration.

- **Message identity/uniqueness guarantees**
  - Unique `MessageID` ensures correctness across retries, replays, and storage backends.

- **Metadata store and storage architecture consolidation**
  - Standardized on etcd for metadata storage
  - Unified shared protobuf/types, improving modularity and future evolvability.

---

## Conclusion

For deeper details on Danube Broker and the broader architecture:

- Documentation: <https://danube-docs.dev-state.com/>
- GitHub repository: <https://github.com/danube-messaging>

Have feature ideas or needs you’d like to see on the platform? Please open a GitHub issue describing your use case and requirements. The roadmap is community-driven, and I welcome contributions and feedback.
