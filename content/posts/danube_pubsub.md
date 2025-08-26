---
title: "Danube: Queuing and Pub/Sub patterns"
date: 2024-08-08
draft: false
categories: ["Danube Messaging"]
author: "Dan Rusei"

autoCollapseToc: true
contentCopyright: MIT

---

[Danube](https://github.com/danube-messaging/danube) is an open-source, distributed publish-subscribe (Pub/Sub) message broker system developed in **Rust**. Danube aims to be a powerful, flexible and scalable messaging solution.

Currently, the Danube platform exclusively supports **Non-persistent messages**. Meaning that the messages reside solely in memory and are promptly distributed to consumers if they are available, utilizing a dispatch mechanism based on subscription types

For comprehensive information on setting up, configuring, and using Danube, please refer to the official [documentation](https://danube-docs.dev-state.com/) and the [previous article](https://dev-state.com/posts/danube_intro/).

## Client Libraries

To interact with the Danube Pub/Sub messaging platform, you can use the Rust or Go client libraries. Each library is designed to provide seamless integration and ease of use.

### Rust Client Library

- **Library**: [danube-client](https://crates.io/crates/danube-client)
- **Description**: An asynchronous Rust client library for Danube.
- **Example Usage**: Explore example usage for producers and consumers on [GitHub repository](https://github.com/danube-messaging/danube-client/tree/main/examples).

### Go Client Library

- **Library**: [danube-go](https://pkg.go.dev/github.com/danrusei/danube-go)
- **Description**: The Go client library for interacting with Danube.
- **Example Usage**: Explore example usage for producers and consumers on [GitHub repository](https://github.com/danube-messaging/danube-go/tree/main/examples).

## Danube Binaries

The [Danube release](https://github.com/danube-messaging/danube/releases) includes several binaries for running, interacting with and managing the Danube platform. Here’s an overview of the available binaries and how to download them:

- **Danube Broker**: The core component of the Danube Pub/Sub platform.
- **Danube Admin**: A command-line interface (CLI) for interacting with and managing the Danube cluster.
- **Danube Pub/Sub CLI**: A CLI tool for handling message publishing and consumption.

 To download the binaries check the [latest release](https://github.com/danube-messaging/danube/releases).

- **Docker Image**: For those who prefer containerized environments.
  - **Docker Image**: `ghcr.io/danube-messaging/danube-broker:v0.1.2`

### danube-pubsub CLI

The `danube-pubsub` CLI allows you to produce and consume messages. Here are some example usages:

- **Produce Messages**: To send messages to a topic, use the `produce` command:

  ```sh
  danube-pubsub produce -s http://127.0.0.1:6650 -c 1000 --message "Hello, Danube!"
  ```

- **Consume Messages**:  To receive messages from a topic, use the consume command:

  ```sh
  danube-pubsub consume -s http://127.0.0.1:6650 --subscription "my-subscription"
  ```

### danube-admin CLI

The `danube-admin` CLI is used for interacting with and managing the Danube cluster:

```sh
danube-admin 
CLI for managing the Danube pub/sub platform

Usage: danube-admin <COMMAND>

Commands:
  brokers     - check Danube cluster brokers info    
  namespaces  - manage and view information about namespaces
  topics      -  manage and view information about topics  

```

## Queuing and Pub/Sub (Fan out) workflows using Danube

This section describes the steps to set up and use Danube for messaging and queuing on Linux.

### Setup Phase

#### Start the ETCD Instance

The etcd instance or cluster is responsible for the metadata persistent storage and cluster synchronization.

To start the ETCD instance, use Docker with the following command:

```sh
docker run -d --name etcd-danube -p 2379:2379 \
    -v /home/rdan/my_projects/danube/./etcd-data:/etcd-data \
    quay.io/coreos/etcd:latest \
    /usr/local/bin/etcd \
    --name etcd-danube \
    --data-dir /etcd-data \
    --advertise-client-urls http://0.0.0.0:2379 \
    --listen-client-urls http://0.0.0.0:2379
```

#### Download and Run the Broker Instance

Download the Danube broker binary and start the broker instance with the following command:

```sh
RUST_LOG=danube_broker=info ./danube-broker --config-file config/danube_broker.yml
```

or RUST_LOG=danube_broker=trace for detailed logging.

### Messaging Queuing Pattern Using a Shared Subscription

**Queueing pattern**: In a **shared subscription** model, messages are placed in a queue and distributed to all consumers subscribing to that topic. When a new consumer is added, the message load is distributed evenly across all consumers, ensuring that each consumer gets a share of the messages.

**Traffic Distribution**: The shared subscription pattern distributes messages in a round-robin fashion. This means that as new consumers join, the system redistributes messages among all active consumers, which helps in balancing the load and improves overall system performance and scalability.

#### Produce Messages

To send messages to a topic, use the produce command:

```sh
danube-pubsub produce -s http://127.0.0.1:6650 -c 1000 -m "Hello, Danube!"

The Message with ID 135 was sent
The Message with ID 136 was sent
...
The Message with ID 148 was sent
```

#### Consume Messages Using a Shared Subscription

To receive messages from a topic with a shared subscription, use the consume command:

```sh
danube-pubsub consume -s http://127.0.0.1:6650 -m my_shared_subscription

Received bytes message: 135, with payload: Hello, Danube!
Received bytes message: 136, with payload: Hello, Danube!
Received bytes message: 137, with payload: Hello, Danube!
Received bytes message: 138, with payload: Hello, Danube!
Received bytes message: 139, with payload: Hello, Danube!
Received bytes message: 140, with payload: Hello, Danube!

!!!! this is where I added the second consumer, notice the message id

Received bytes message: 142, with payload: Hello, Danube!
Received bytes message: 144, with payload: Hello, Danube!
Received bytes message: 146, with payload: Hello, Danube!
Received bytes message: 148, with payload: Hello, Danube!
```

#### Add Another Consumer to the Shared Subscription

In a separate shell, add another consumer to the shared subscription:

```sh
danube-pubsub consume -s http://127.0.0.1:6650 -m my_shared_subscription -c other2_consumer

Received bytes message: 141, with payload: Hello, Danube!
Received bytes message: 143, with payload: Hello, Danube!
Received bytes message: 145, with payload: Hello, Danube!
Received bytes message: 147, with payload: Hello, Danube!
```

#### Notice on Consumer Behavior

When the second consumer was introduced, you can observe that the messages are distributed in a round-robin fashion between the consumers. This distribution method ensures that messages are balanced across all available consumers, which helps in efficiently handling message traffic and improves scalability.

### Messaging Pub/Sub (Fan-Out) Pattern

**Pub/Sub pattern**: The pub/sub (publish/subscribe) pattern allows multiple consumers to receive messages from a single topic, where the publisher sends messages to the topic without knowing who will receive them. Subscribers to the topic receive a copy of each message, enabling broad distribution and real-time updates across various consumers.

In Danube, by using **multiple exclusive subscriptions**, you can emulate the pub-sub pattern by effectively isolating different consumers. Each consumer gets its own dedicated stream of messages, allowing for independent processing while still benefiting from the message distribution capabilities of the system.

#### Produce Messages

To send messages to a topic using the fan-out pattern, use the following command:

```sh
danube-pubsub produce -s http://127.0.0.1:6650 -c 1000 -m "Hello, Danube!"
```

#### Consume Messages

To receive messages with an exclusive subscription, use the consume command with the --sub-type exclusive option:

```sh
danube-pubsub consume -s http://127.0.0.1:6650 -m my_exclusive --sub-type exclusive

Received bytes message: 37, with payload: Hello, Danube!
Received bytes message: 38, with payload: Hello, Danube!
Received bytes message: 39, with payload: Hello, Danube!
Received bytes message: 40, with payload: Hello, Danube!
```

#### Handling Exclusive Subscriptions

When using exclusive subscriptions, only one consumer is allowed per subscription. If you try to add a second consumer to the same exclusive subscription, you will encounter the following error:

```sh
danube-pubsub consume -s http://127.0.0.1:6650 -m my_exclusive -c other2_consumer --sub-type exclusive

Error: from status error: status: PermissionDenied, message: "Not allowed to add the Consumer: other2_consumer, the Exclusive subscription can't be shared with other consumers"

```

This error occurs because an exclusive subscription is designed to be consumed by only one consumer at a time. To add additional consumers, you need to create a new exclusive subscription with a different name:

```sh
danube-pubsub consume -s http://127.0.0.1:6650 -m my_exclusive2 --sub-type exclusive

Received bytes message: 372, with payload: Hello, Danube!
Received bytes message: 373, with payload: Hello, Danube!
Received bytes message: 374, with payload: Hello, Danube!
Received bytes message: 375, with payload: Hello, Danube!
Received bytes message: 376, with payload: Hello, Danube!
```

## Summary

In summary, **Danube** offers an robust and efficient pub/sub messaging platform . With flexible subscription models, straightforward setup, it enables scalable, real-time messaging for diverse applications.
