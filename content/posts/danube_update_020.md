---
title: "Danube Updates v0.2.0"
date: 2025-01-01
draft: false
categories: ["Danube Messaging"]
author: "Dan Rusei"

autoCollapseToc: true
contentCopyright: MIT

---

[Danube Messaging](https://github.com/danube-messaging/danube) is an open-source, distributed publish-subscribe (Pub/Sub) message broker system developed in Rust. Danube aims to be a powerful, flexible and scalable messaging solution. Allows single or multiple Producers to publish on the Topics and multiple Subscriptions to consume the messages from the Topic.

This article aims to cover the latest developments on the Danube platform. Before that, here is a summary of Danube's capabilities. Since Danube is still under heavy development, some of these capabilities may need further refinement to improve performance or ensure all corner cases are supported.

* **Topics**: A unit of storage that organizes messages into a stream.
  * **Non-partitioned topics**: Served by a single broker.
  * **Partitioned topics**: Divided into partitions, served by different brokers within the cluster, enhancing scalability and fault tolerance.
* **Message Dispatch**:
  * **Non-reliable Message Dispatch**: Messages reside in memory and are promptly distributed to consumers, ideal for scenarios where speed is crucial.
  * **Reliable Message Dispatch**: Supports configurable storage options including in-memory, disk, and S3, ensuring message persistence and durability.
* **Metadata Store**:
  * **ETCD as Default**: Provides a reliable and consistent Metadata store for cluster synchronization.
  * **Configurable Options**: Allows customization of metadata storage to fit specific requirements.
* **Subscription Types**:
  * Supports various subscription types (exclusive, shared, failover) enabling different messaging patterns such as message queueing and pub-sub.
* **Flexible Message Schemas**
  * Supports multiple message schemas (bytes, string, int64, JSON) providing flexibility in message format and structure.
* **Command-Line Interfaces (CLI)**
  * **Danube CLI**: For handling message publishing and consumption.
  * **Danube Admin CLI**: For managing and interacting with the Danube cluster, including broker, namespace, and topic management.

## Danube v0.2.0 main updates

The most significant updates in this version include the addition of Reliable Message Dispatch and the introduction of the danube-metadata-store library. This new library aims to reduce the hard dependency on ETCD for metadata storage by providing an abstraction layer that allows different storage systems to be used interchangeably for managing metadata. It offers a unified interface for operations such as get, put, and watch across various backend implementations.

### Reliable Message Dispatch in Danube

The reliable message dispatch feature in Danube ensures that messages are stored and delivered reliably, even in the event of failures. This is achieved using an abstraction layer `StorageBackend` trait that allows different storage systems to be used interchangeably for storing messages. The aim is to use different systems such as in-memory, disk, or S3, providing flexibility in how messages are stored and retrieved.

#### Core Components

* [**TopicStore**](https://github.com/danube-messaging/danube/blob/main/danube-reliable-dispatch/src/topic_storage.rs#L35): Manages the storage of messages in a queue for reliable delivery. The smallest unit of storage is Segment, and the role of the TopicStore is to manage these Segments by storing new messages, stoging and retrieving from persistent storage backends
* [**SubscriptionDispatch**](https://github.com/danube-messaging/danube/blob/main/danube-reliable-dispatch/src/dispatch.rs#L17): Handles the delivery of messages to consumers, ensuring reliable dispatch based on the last acknowledged message.

#### Example usage

This example sets up a reliable dispatch producer in Danube. In the following sequence of lines, the client is constructed and the ConfigDispatchStrategy is added in the definition of the producer. Although the producer is dispatch agnostic, the topic is created along with the producer if it doesn't already exist.

```rust
    let client = DanubeClient::builder()
        .service_url("http://127.0.0.1:6650")
        .build()
        .unwrap();

    let topic = "/default/reliable_topic";
    let producer_name = "prod_json_reliable";

    let storage_type = StorageType::InMemory;
    let reliable_options = ReliableOptions::new(
        5, // segment size in MB
        storage_type,
        RetentionPolicy::RetainUntilExpire,
        3600, // 1 hour
    );
    let dispatch_strategy = ConfigDispatchStrategy::Reliable(reliable_options);

    let mut producer = client
        .new_producer()
        .with_topic(topic)
        .with_name(producer_name)
        .with_dispatch_strategy(dispatch_strategy)
        .build();

    producer.create().await?;
```

### Metadata Storage in Danube

Up until version 0.2.0, the ETCD logic was embedded directly into the Danube broker. To modularize and improve the architecture, a new crate called [`danube-metadata-store`](https://github.com/danube-messaging/danube/tree/main/danube-metadata-store) was created. This crate abstracts the metadata storage logic, allowing different storage systems to be used interchangeably. This modular approach ensures that Danube can adapt to different infrastructure requirements while maintaining a clean and maintainable codebase.

#### Core Components

* [**MetadataStorage**](https://github.com/danube-messaging/danube/blob/main/danube-metadata-store/src/lib.rs#L24): This enum defines various storage backends that can be used for metadata storage, such as ETCD, Zookeper, and an in-memory store for testing purposes. It provides a unified interface for operations like get, put, delete, and watch.
* [**MetadataStore**](https://github.com/danube-messaging/danube/blob/main/danube-metadata-store/src/store.rs#L15): This trait defines the core operations for managing metadata. Implementations of this trait for different storage backends ensure that the same interface can be used regardless of the underlying storage system.

### What's Next for Danube

Up until now, the main effort has been to put together the core components of the Danube messaging platform. In the upcoming period, I will refrain from adding more features and instead focus on improving testing for newly added features, resolving pending issues, and enhancing performance and code reliability.

If you would like to contribute, you are more than welcome to [take on an issue](https://github.com/danube-messaging/danube/issues) from the Danube repository, suggest any features that are important for your use case, or report any bugs. Additionally, as performance and reliability are critical, your help is greatly appreciated by raising PRs with your suggestions.

If you like [the Danube project](https://github.com/danube-messaging/danube) or you think is somehow valuable give it a github like.

In the mid-term, I plan to create integrations and source and sink connectors with major platforms to allow easy pulling and consuming of data from Danube topics.
