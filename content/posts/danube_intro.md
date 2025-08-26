---
title: "Danube - Pub-Sub message broker"
date: 2024-06-29
draft: false
categories: ["Danube Messaging"]
author: "Dan Rusei"

autoCollapseToc: true
contentCopyright: MIT

---

[Danube](https://github.com/danube-messaging/danube) is an open-source, distributed publish-subscribe (Pub/Sub) message broker system developed in Rust. Inspired by the Apache Pulsar messaging and streaming platform, Danube incorporates some similar concepts but is designed to carve its own path within the distributed messaging ecosystem.

At the time of writing, the Danube platform is in the early stages of development and may have missing or incomplete functionalities. Use with caution. Contributions are welcome, and you can also report any issues you encounter.

Currently, the Danube platform exclusively supports Non-persistent messages. Meaning that  the messages reside solely in memory and are promptly distributed to consumers if they are available, utilizing a dispatch mechanism based on subscription types.

## Danube Architecture

***This is old architecture diagram, for better understanding of the Danube architecture check the [Danube Docs](https://danube-docs.dev-state.com/architecture/architecture/).***

Danube is a distributed platform that relies on ETCD as a persistent metadata storage system. This setup ensures all metadata created during system operations is reliably stored. The Danube cluster consists of one or more Danube brokers, which are stateless message brokers. This stateless design allows cluster administrators to easily spin up new instances, as topics are automatically distributed across available instances. Producers connect to the brokers to publish messages, while consumers connect to the brokers to consume messages. The communication between producers/consumers and Danube brokers is based on GRPC.

{{< image src="/img/2024/Danube_architecture_non_persistent.png" style="border-radius: 8px;" >}}

### Danube Cluster Services Role

Danube relies on several components, to enable the distributed behavior.

#### External Service - ETCD Metadata Storage

This is the Metadata Storage responsible for the persistent storage of metadata and cluster synchronization.

#### Danube Service Components

**Broker Service** - The Broker Service owns the topics and manages their lifecycle. It also facilitates the creation of producers, subscriptions, and consumers, ensuring that producers can publish messages to topics and consumers can consume messages from topics.

**Leader Election Service** - The Leader Election Service selects one broker from the cluster to act as the Leader. The Broker Leader is responsible for making decisions. This service is used by the Load Manager, ensuring only one broker in the cluster posts the cluster aggregated Load Report.

**Load Manager Service** - The Load Manager monitors and distributes load across brokers by managing topic and partition assignments. It implements rebalancing logic to redistribute topics/partitions when brokers join or leave the cluster and is responsible for failover mechanisms to handle broker failures.

**Local Metadata Cache** - This cache stores various types of metadata required by Danube brokers, such as topic and namespace data, which are frequently accessed during message production and consumption. This reduces the need for frequent queries to the central metadata store, ETCD.

**Syncronizer** - Yet to be implemented, the synchronizer ensures that metadata and configuration settings across different brokers remain consistent. It propagates changes to metadata and configuration settings using client Producers and Consumers.

For additional details on each service components [check out this document](https://github.com/danrusei/danube/blob/main/docs/internal_danube_services.md).

## Producers / Consumers

Before an application creates a producer/consumer, the client library needs to initiate a setup phase including two steps:

* The client attempts to determine the owner of the topic by sending a Lookup request to Broker.
* Once the client library has the broker address, it creates a RPC connection (or reuses an existing connection from the pool) and (in later stage authenticates it ).
* Within this connection, the clients (producer, consumer) and brokers exchange RPC commands. At this point, the client sends a command to create producer/consumer to the broker, which will comply after doing some validation checks.

{{< image src="/img/2024/producers_consumers.png" style="border-radius: 8px;" >}}

### Producer

A producer is a process that attaches to a topic and publishes messages to a Danube broker. The Danube broker processes the messages.

**Access Mode** is a mechanism to determin the permissions of producers on topics.

* **Shared** - Multiple producers can publish on a topic.
* **Exclusive** - If there is already a producer connected, other producers trying to publish on this topic get errors immediately

### Consumer

A consumer is a process that attaches to a topic via a subscription and then receives messages.

**Subscription Types** - describe the way the consumers receive the messages from topics

* **Exclusive** - Only one consumer can subscribe, guaranteeing message order.
* **Shared** - Multiple consumers can subscribe, messages are delivered round-robin, offering good scalability but no order guarantee.
* **Failover** - Similar to shared subscriptions, but multiple consumers can subscribe, and one actively receives messages.

## Demo time

I'm using the development environment to showcase the Danube functionality. In order to replicate the steps below, first need to clone the [Danube repository](https://github.com/danube-messaging/danube) locally.

### Start the ETCD instance

{{< code language="bash" isCollapsed="false" >}}
make etcd
Starting ETCD...
docker run -d --name etcd-danube -p 2379:2379 \
    -v /home/rdan/my_projects/danube/./etcd-data:/etcd-data \
    quay.io/coreos/etcd:latest \
    /usr/local/bin/etcd \
    --name etcd-danube \
    --data-dir /etcd-data \
    --advertise-client-urls <http://0.0.0.0:2379> \
    --listen-client-urls <http://0.0.0.0:2379>
4ae43909314e6764b8938eccbb9271bbfcad13111620cd879844641b0098f3d6
ETCD instance started on port: 2379
{{< /code >}}

### Run 3 Message Brokers in the Danube Cluster

{{< code language="bash" isCollapsed="false" >}}
cargo build
cargo build --examples

make brokers RUST_LOG=danube_broker=info
{{< /code >}}

### Create the Producer and publish the messages

For complete code check the [producer.rs example](https://github.com/danube-messaging/danube/blob/main/danube-client/examples/producer.rs)

* **Create the DanubeClient**:

{{< code language="rust" isCollapsed="false" >}}
let client = DanubeClient::builder()
        .service_url("http://[::1]:6650")
        .build()
        .unwrap();
{{< /code >}}

* **Create the Producer**:

{{< code language="rust" isCollapsed="false" >}}
let topic = "/default/test_topic".to_string();

    let json_schema = r#"{"type": "object", "properties": {"field1": {"type": "string"}, "field2": {"type": "integer"}}}"#.to_string();

    let mut producer = client
        .new_producer()
        .with_topic(topic)
        .with_name("test_producer1")
        .with_schema("my_app".into(), SchemaType::Json(json_schema))
        .build();

    let prod_id = producer.create().await?;
    info!("The Producer was created with ID: {:?}", prod_id);
{{< /code >}}

* **Send the messages**

{{< code language="rust" isCollapsed="false" >}}
while i < 20 {
        let data = json!({
            "field1": format!{"value{}", i},
            "field2": 2020+i,
        });

        // Convert to string and encode to bytes
        let json_string = serde_json::to_string(&data).unwrap();
        let encoded_data = json_string.as_bytes().to_vec();

        // let json_message = r#"{"field1": "value", "field2": 123}"#.as_bytes().to_vec();
        let message_id = producer.send(encoded_data).await?;
        println!("The Message with id {} was sent", message_id);

        thread::sleep(Duration::from_secs(1));
        i += 1;
    }
{{< /code >}}

Run the producer example

{{< code language="bash" isCollapsed="false" >}}
RUST_LOG=producer=info target/debug/examples/producer
{{< /code >}}

### Create the Consumer and consume the messages

For complete code check the [consumer.rs example](https://github.com/danrusei/danube/blob/main/danube-client/examples/consumer.rs)

* **Create the DanubeClient**:

{{< code language="rust" isCollapsed="false" >}}
let client = DanubeClient::builder()
        .service_url("http://[::1]:6650")
        .build()
        .unwrap();
{{< /code >}}

* **Create the Consumer and subscribe to a subscription**:

{{< code language="rust" isCollapsed="false" >}}
let topic = "/default/test_topic".to_string();

    let mut consumer = client
        .new_consumer()
        .with_topic(topic.clone())
        .with_consumer_name("test_consumer")
        .with_subscription("test_subscription")
        .with_subscription_type(SubType::Exclusive)
        .build();

    // Subscribe to the topic
    let consumer_id = consumer.subscribe().await?;
    println!("The Consumer with ID: {:?} was created", consumer_id);
{{< /code >}}

* **Consume the messages**

{{< code language="rust" isCollapsed="false" >}}
// Start receiving messages
    let mut message_stream = consumer.receive().await?;

    while let Some(message) = message_stream.next().await {
        match message {
            Ok(stream_message) => {
                let payload = stream_message.messages;
                // Deserialize the message using the schema
                match serde_json::from_slice::<MyMessage>(&payload) {
                    Ok(decoded_message) => {
                        println!("Received message: {:?}", decoded_message);
                    }
                    Err(e) => {
                        eprintln!("Failed to decode message: {}", e);
                    }
                }
            }
            Err(e) => {
                eprintln!("Error receiving message: {}", e);
                break;
            }
        }
    }
{{< /code >}}

Run the consumer example:

{{< code language="bash" isCollapsed="false" >}}
RUST_LOG=producer=info target/debug/examples/consumer
{{< /code >}}

### Check the Brokers logs

Below are the info logs of 2 brokers part of the Danube cluster. These are relevant to showcase the distributed behavior. The broker that receive the request is the Leader broker and assign the topic to another broker in the cluster.

The Leader broker:

{{< code language="bash" isCollapsed="false" >}}
tail -f temp/broker_6650.log

INFO danube_broker: Use ETCD storage as metadata persistent store
INFO danube_broker: Start the Danube Service
INFO danube_broker::danube_service: Setting up the cluster MY_CLUSTER
INFO danube_broker::danube_service::broker_register: Broker 3320012749120699522 registered in the cluster
INFO danube_broker::danube_service: cluster metadata setup completed
INFO danube_broker::danube_service:  Started the Broker GRPC server
INFO danube_broker::broker_server: Server is listening on address: [::1]:6650
INFO danube_broker::danube_service: Started the Leader Election service
INFO danube_broker::danube_service::local_cache: Initial cache populated
INFO danube_broker::danube_service: Started the Local Cache service.
INFO danube_broker::danube_service: Started the Load Manager service.
INFO create_producer: danube_broker::broker_server: New Producer request with name: test_producer1 for topic: /default/test_topic
INFO danube_broker::danube_service::load_manager: Attempting to assign the new topic /cluster/unassigned/default/test_topic to a broker
INFO danube_broker::danube_service::load_manager: The topic /default/test_topic was successfully assign to broker 12706277172540671034
INFO create_producer: danube_broker::broker_server: Error topic request: The topic metadata was created, need to redo the lookup to find the correct broker
{{< /code >}}

Another Broker in the cluster that is notified that should host the topic and serve the producer and consumer.

{{< code language="bash" isCollapsed="false" >}}
tail -f temp/broker_6652.log

INFO danube_broker: Use ETCD storage as metadata persistent store
INFO danube_broker: Start the Danube Service
INFO danube_broker::danube_service: Setting up the cluster MY_CLUSTER
INFO danube_broker::danube_service::broker_register: Broker 12706277172540671034 registered in the cluster
INFO danube_broker::danube_service: cluster metadata setup completed
INFO danube_broker::danube_service:  Started the Broker GRPC server
INFO danube_broker::broker_server: Server is listening on address: [::1]:6652
INFO danube_broker::danube_service: Started the Leader Election service
INFO danube_broker::danube_service::local_cache: Initial cache populated
INFO danube_broker::danube_service: Started the Local Cache service.
INFO danube_broker::danube_service: Started the Load Manager service.
INFO danube_broker::danube_service: A new Watch event has been received ETCDWatchEvent { key: "/cluster/brokers/12706277172540671034/default/test_topic", value: Some([110, 117, 108, 108]), mod_revision: 24, version: 1, event_type: Put }
INFO danube_broker::danube_service: The topic /default/test_topic , was successfully created on broker 12706277172540671034
INFO create_producer: danube_broker::broker_server: New Producer request with name: test_producer1 for topic: /default/test_topic
INFO create_producer: danube_broker::broker_server: topic_name: /default/test_topic was found
INFO create_producer: danube_broker::broker_server: The Producer with name: test_producer1 and with id: 9827454625296160679, has been created
INFO subscribe: danube_broker::broker_server: New Consumer request with name: test_consumer for topic: /default/test_topic with subscription_type 1
INFO subscribe: danube_broker::broker_server: topic_name: /default/test_topic was found
INFO subscribe: danube_broker::broker_server: The Consumer with id: 17353146059293792005 for subscription: test_subscription, has been created.
INFO receive_messages: danube_broker::broker_server: The Consumer with id: 17353146059293792005 requested to receive messages
{{< /code >}}

## Conclusion

The Danube broker messaging platform is currently under active development, which means the API may undergo slight changes over time to accommodate all use cases. This ongoing development aims to enhance the feature set & reliability, ensuring it can meet the diverse needs of the users.

Contributions are welcome, and you can report any issues you encounter. If you find this project interesting or are interested in its future development, give it a [GitHub star](https://github.com/danube-messaging/danube).

The client library is currently written in Rust, with a Go client potentially coming soon. Contributions in other languages, such as Python, Java, etc., are also greatly appreciated.
