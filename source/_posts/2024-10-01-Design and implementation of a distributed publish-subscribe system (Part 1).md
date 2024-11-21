---
layout: single
---

## Introduction

A distributed message subscription system is an architecture for processing and distributing messages, aiming to achieve efficient and reliable information delivery. The system consists of three main components: broker, publisher, and subscriber.

## Main components

### 1. Broker

The broker is the core of the distributed message subscription system, responsible for managing topics, subscriptions, and message distribution. Broker nodes are interconnected to form a network, managing topic creation, topic lists, subscriber lists, and message routing to ensure that messages can be effectively delivered between different topics and subscribers.

### 2. Publisher

The publisher is a client system responsible for creating topics and publishing messages to topics. The publisher can be any application or service that can send messages. Specific functions of the publisher:

1. Create a new topic: Generate a unique topic ID (such as UUID) and assign a name (not necessarily unique, because multiple publishers may have topics with the same name).

2. Publish a message to an existing Topic: Send a message through the Broker of the Topic, using a unique topic ID. The message should be sent to all topic subscribers. Each message will be limited to a maximum of 100 characters. There is no need to retain messages in any broker.

3. Show subscriber count: Display the total number of subscribers for each topic associated with this.

4. Delete topic: Delete the topic from the system and automatically unsubscribe all currently subscribing scribes. A notification message should be sent to each subscriber.

### 3. Subscribers

Subscribers are clients who express interest by subscribing to specific topics through brokers. They receive real-time messages about these topics from broker nodes. Subscriber specific functions:

1. List all available topics: Retrieve a list of all available topics in the broker network, including topic ID, topic name, and publisher name.

2. Subscribe to a topic: Subscribe to a topic using its unique ID. The subscriber will receive all future messages about this topic.

3. Show current subscriptions: List active subscriptions, including topic ID, topic name, and publisher name.

4. Unsubscribe from a topic: Stop receiving messages from a topic. The broker sends a notification to confirm the unsubscription message

![Figure1]({{ site.baseurl }}/assets/images/broker.jpg)

## Design ideas

1. Whether it is the communication between nodes and publishers, subscribers, or the communication between nodes, it is based on sockets.

![Figure2]({{ site.baseurl }}/assets/images/socket.jpg)

2. Create a thread to realize the communication between message subscribers and publishers. Create a topic class to store subscribers to ensure that when the publisher publishes a message to the topic, it can notify the subscribers under the topic.

3. Create another thread to realize the interconnection of multiple nodes. When a node receives an instruction, ensure that other nodes also receive the notification to ensure the synchronization of the message.

Based on the above ideas, we gradually implement this system. The specific implementation will be introduced in the next article.