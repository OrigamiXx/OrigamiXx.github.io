---
layout: single
---
## System Architecture Overview

In the first two articles, we introduced the basic concepts of the message subscription system and its implementation under a single node. As the system scale expands and performance requirements increase, the single-node architecture obviously cannot meet the requirements of high concurrency and high availability. This article will explore how to implement publishers, subscribers, and brokers in a distributed environment and solve the challenges faced in the distributed architecture.

1. The publisher publishes the message to the specified topic.
2. After the broker receives the message, it finds the corresponding subscriber based on the topic.
3. The broker pushes the message to the subscribers to whom it is connected and subscribes to the topic.
4. The broker synchronizes the message to other brokers.
5. Other brokers push the message to the subscribers to whom they are connected and subscribe to the topic.

## Implementation details

### 1. Receive connections from other brokers
Use type to distinguish whether it is a connection from other brokers, a connection from a publisher, or a connection from a subscriber. If it is a connection from other brokers, use BrokerHandler to handle it. If it is a connection from a client, use ClientHandler to handle it.
```java
        // Parse command line arguments
        for (int i = 1; i < args.length; i++) {
            if ("-b".equals(args[i])) {

                StringBuilder brokers = new StringBuilder();
                for (int j = i + 1; j < args.length; j++) {
                    brokers.append(args[j]).append(" ");
                }
                brokersArg = brokers.toString().trim();
                break;
            }
        }

        if (!brokersArg.isEmpty()) {

            connectToOtherBrokers(brokersArg);
            new Thread(new BrokerConnectionListener(args[2])).start();
        }


        // Accept connections from clients and other Brokers
        while (true) {
            Socket socket = serverSocket.accept();
            new Thread(() -> {
                try {
                    BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
                    PrintWriter out = new PrintWriter(socket.getOutputStream(), true);


                    String type = in.readLine();
                    if ("BROKER".equalsIgnoreCase(type)) {

                        brokerConnections.add(socket);
                        new Thread(new BrokerHandler(socket)).start();
                        System.out.println("Accepted connection from another broker.");
                    } else if ("SUB".equalsIgnoreCase(type) || "PUB".equalsIgnoreCase(type)) {

//                        clientConnections.add(socket);
                        new Thread(new ClientHandler(socket)).start();
                        System.out.println("Accepted connection from a client.");


                        if ("SUB".equalsIgnoreCase(type)) {
                            subscriberCount++;
                            if (subscriberCount > MAX_SUB) {
                                //socket.close();
                                sendResponse(socket, "close");
                                subscriberCount--;
                            }

                        } else {
                            publisherCount++;
                            if (publisherCount > MAX_PUB) {
                                //socket.close();
                                sendResponse(socket, "close");
                                publisherCount--;
                            }

                        }

                    }
                } catch (IOException e) {
                    //e.printStackTrace();
                    System.out.println(e.getMessage());
                }
            }).start();
        }

```
### 2. Connect to other brokers
```java
 
      private static void connectToOtherBrokers(String brokersArg) {
        String[] brokers = brokersArg.split(" ");
        for (String broker : brokers) {
            String[] brokerInfo = broker.split(":");
            String brokerIp = brokerInfo[0];
            int brokerPort = Integer.parseInt(brokerInfo[1]);
            try {
                Socket brokerSocket = new Socket(brokerIp, brokerPort);
                PrintWriter out = new PrintWriter(brokerSocket.getOutputStream(), true);
                out.println("BROKER");
                brokerConnections.add(brokerSocket);
                new Thread(new BrokerHandler(brokerSocket)).start();
                System.out.println("Connected to broker: " + brokerIp + ":" + brokerPort);
            } catch (IOException e) {
                System.out.println("Failed to connect to broker: " + brokerIp + ":" + brokerPort);
                if (!failedBrokers.contains(broker)) {
                    failedBrokers.add(broker);
                }

            }
        }
    }

```
### 3. Broker connection listener
During the broker operation, new proxies may be added. Create a broker listener to monitor.
```java
// Listener class: Periodically retry the Broker that failed to connect
    private static class BrokerConnectionListener implements Runnable {
        private final String brokersArg;
        private static final int RETRY_INTERVAL = 5000;

        public BrokerConnectionListener(String brokersArg) {
            this.brokersArg = brokersArg;
        }

        @Override
        public void run() {
            while (true) {
                try {
                    Thread.sleep(RETRY_INTERVAL);
                    retryFailedConnections();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }

        // Retry the Broker that failed to connect
        private void retryFailedConnections() {
            Iterator<String> iterator = failedBrokers.iterator();
            while (iterator.hasNext()) {
                String broker = iterator.next();
                String[] brokerInfo = broker.split(":");
                String brokerIp = brokerInfo[0];
                int brokerPort = Integer.parseInt(brokerInfo[1]);
                try {
                    Socket brokerSocket = new Socket(brokerIp, brokerPort);
                    brokerConnections.add(brokerSocket);
                    new Thread(new BrokerHandler(brokerSocket)).start();
                    System.out.println("Reconnected to broker: " + brokerIp + ":" + brokerPort);
                    iterator.remove();
                } catch (IOException e) {
                    System.out.println("Retrying failed broker: " + brokerIp + ":" + brokerPort);
                }
            }
        }
    }

```

### 4. Synchronize messages to other brokers
When processing publisher-subscriber messages of this broker node, it is necessary to synchronize them to other nodes.
```java

 //Broadcast messages to other brokers
    private static void broadcastToBrokers(String message) {
        for (Socket brokerSocket : brokerConnections) {
            try {
                PrintWriter out = new PrintWriter(brokerSocket.getOutputStream(), true);
                out.println(message);
            } catch (IOException e) {
                System.out.println("Failed to broadcast message to broker: " + e.getMessage());
            }
        }
    }

```


### 5. Receive and process messages from other brokers
Parse messages from other brokers and process them accordingly, such as synchronously creating topics and synchronously subscribing.
```java
// Used to process messages from other Brokers
    private static class BrokerHandler implements Runnable {
        private final Socket brokerSocket;

        public BrokerHandler(Socket brokerSocket) {
            this.brokerSocket = brokerSocket;
        }

        @Override
        public void run() {
            try (BufferedReader in = new BufferedReader(new InputStreamReader(brokerSocket.getInputStream()))) {
                String message;
                while ((message = in.readLine()) != null) {
                    handleBrokerMessage(message);
                }
            } catch (IOException e) {
                //e.printStackTrace();
                System.out.println(e.getMessage());
            }
        }


        // Process messages from other Brokers
        private static void handleBrokerMessage(String message) {
            String[] parts = message.split(" ");
            String command = parts[0];


            switch (command) {
                case "CREATE":
                    createTopic(parts);
                    break;
                case "PUBLISH":
                    publishMessage(parts);
                    break;
                case "SUBSCRIBE":
                    subscribe(parts);
                    break;
                case "DELETE":
                    deleteTopic(parts);
                    break;
                case "UNSUBSCRIBE":
                    unsubscribe(parts);
                default:
                    break;
            }


        }


        ......
    }


```


## Test
Start two brokers broker1 and broker2 in sequence. Broker1 connects to publisher and subscriber1; broker2 connects to subscriber2; subscriber1 and subscriber2 subscribe to the topic created by publisher.

Publisher creates a topic and sends a message
![Figure1]({{ site.baseurl }}/assets/images/ts1.jpg)


Subscribers on two different nodes receive the message
![Figure2]({{ site.baseurl }}/assets/images/ts2.jpg)
![Figure3]({{ site.baseurl }}/assets/images/ts3.jpg)
