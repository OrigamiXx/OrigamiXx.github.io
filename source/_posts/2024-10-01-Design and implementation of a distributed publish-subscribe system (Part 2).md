---
layout: single
---
## System Architecture Overview

In a single-node environment, the broker is responsible for coordinating the interaction between publishers and subscribers, storing messages and pushing messages to subscribed clients. The overall process is as follows:

1. The publisher publishes the message to the specified topic.

2. After receiving the message, the broker finds the corresponding subscriber based on the topic.

3. The broker pushes the message to all subscribers who have subscribed to the topic.

## Implementation details

### 1. Publisher Implementation

The publisher is a client system that is responsible for creating topics and publishing messages to topics. The publisher can be any application or service that can send messages.

The publisher creates a socket connection with the broker through new Socket. It writes messages to the broker through PrintWriter and reads messages from the broker through BufferedReader.
```java

// Create a Socket connected to the Broker
        // Create a Socket connected to the Broker
        Socket socket = new Socket(brokerIp, brokerPort);
        PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));
        BufferedReader brokerReader = new BufferedReader(new InputStreamReader(socket.getInputStream()));


        out.println("PUB");


        new Thread(() -> {
            try {
                String response;

                while ((response = brokerReader.readLine()) != null) {
                    System.out.println("[Response from Broker]: " + response);
                    if (response.equals("close")){
                        socket.close();
                    }
                }
                System.out.println("res:"+brokerReader.readLine());
            } catch (IOException e) {
                if (e.getMessage().equals("socket closed")){

                    System.exit(0);
                }

                //e.printStackTrace();
            }
        }).start();


```
The publisher creates a topic, combines the topic ID, topic name, and publisher username into a string, and sends it to the broker through the socket for processing.
```java
 private static void createTopic(String[] parts, PrintWriter out) {
        if (parts.length != 3) {
            System.out.println("[ERROR] Parameter error.");
            return;
        }

        String topicId = parts[1];
        String topicName = parts[2];
        String message = "CREATE " + topicId + " " + topicName + " " + username;

        out.println(message);
    }

```
Publisher sends message
```java

 private static void publishMessage(String[] parts, PrintWriter out) {
        if (parts.length < 3) {
            System.out.println("[ERROR] Parameter error.");
            return;
        }

        String topicId = parts[1];
        String content = String.join(" ", Arrays.copyOfRange(parts, 2, parts.length));

        if(content.length()>100){
            System.out.println("[ERROR] No more than 100 characters.");
            return;
        }

        String message = "PUBLISH " + topicId + " " + content+ " " +username;

        out.println(message);
    }

```
### 2.Subscriber Implementation

Subscribers are clients that express interest in specific topics by subscribing to them through the broker. They receive real-time messages about these topics from the broker node.

Subscribers also create a socket connection to the broker through new Socket. They write messages to the broker through PrintWriter and read messages from the broker through BufferedReader.

```java

        Socket socket = new Socket(brokerIp, brokerPort);
        PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
        BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));
        BufferedReader brokerReader = new BufferedReader(new InputStreamReader(socket.getInputStream()));


        out.println("SUB");


        new Thread(() -> {
            try {
                String message;
                while ((message = brokerReader.readLine()) != null) {
                    System.out.println("[Response from Broker]: " + message);
                    if (message.equals("close")){
                        socket.close();
                    }
                }
            } catch (IOException e) {
                if (e.getMessage().equals("socket closed")){

                    System.exit(0);
                }
            }
        }).start();


```
When a subscriber subscribes to a topic, the topic ID and the subscriber's username are combined into a message and sent to the broker.

```java
 private static void subscribe(String[] parts, PrintWriter out) {
        if (parts.length != 2) {
            System.out.println("[ERROR] Parameter error.");
            return;
        }

        String topicId = parts[1];

        String message = "SUBSCRIBE " + topicId + " " + username;


        out.println(message);
    }

```



### 3.Broker Implementation

The broker is the core of the distributed message subscription system, responsible for managing topics, subscriptions, and message distribution. The broker nodes are connected to each other to form a network, managing topic creation, topic lists, subscriber lists, and message routing, ensuring that messages can be effectively delivered between different topics and subscribers.

Under a single node, the broker only needs to receive messages from subscribers or publishers and process them accordingly, without considering the message synchronization issues between multiple nodes.

Get the socket from the publisher or subscriber through serverSocket.accept().
```java

       ServerSocket serverSocket = new ServerSocket(port);
        System.out.println("Broker started on port: " + port);



        while (true) {
            Socket socket = serverSocket.accept();
            new Thread(() -> {
                try {
                    BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
                    PrintWriter out = new PrintWriter(socket.getOutputStream(), true);


                    String type = in.readLine();
                    if ("SUB".equalsIgnoreCase(type) || "PUB".equalsIgnoreCase(type)) {

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
Create a ClientHandler that implements the Runnable interface to process various messages from publishers or subscribers in the child thread
```java

private static class ClientHandler implements Runnable {
        private final Socket clientSocket;

        public ClientHandler(Socket socket) {
            this.clientSocket = socket;
        }

        @Override
        public void run() {
            try (BufferedReader in = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
                 PrintWriter out = new PrintWriter(clientSocket.getOutputStream(), true)) {

                String message;
                while ((message = in.readLine()) != null) {
                    handleClientMessage(message, clientSocket);
                }
            } catch (IOException e) {
                //e.printStackTrace();
                System.out.println(e.getMessage());
            }
        }


        private void handleClientMessage(String message, Socket socket) {
            String[] parts = message.split(" ");
            String command = parts[0];

            switch (command) {
                case "CREATE":
                    createTopic(parts, socket);
                    
                    break;
                case "PUBLISH":
                    publishMessage(parts, socket);
                    
                    break;
                case "SHOW":
                    showSubscribers(parts, socket);
                    break;
                case "DELETE":
                    deleteTopic(parts, socket);
                    
                    break;
                case "SUBSCRIBE":
                    subscribe(parts, socket);
                    
                    break;
                case "DISPLAY":
                    displayTopics(socket);
                    break;
                case "CURRENT":
                    showCurrentSubscriptions(parts,socket);
                    break;
                case "UNSUBSCRIBE":
                    unsubscribe(parts, socket);
                    
                    break;
                default:
                    System.out.println("[ERROR] Illegal client instruction.");
            }
        }

        ......


    }

```

## Test

Start the broker, subscriber, and publisher in sequence. First, the publisher creates a topic, the subscriber subscribes to the topic, and then the publisher publishes a message, and the subscriber successfully receives the message. In this way, the publisher-subscriber function under a single node has been implemented. We will implement a publish-subscribe system under multiple nodes in the next article.
![Figure1]({{ site.baseurl }}/assets/images/ss1.jpg)
![Figure2]({{ site.baseurl }}/assets/images/ss2.jpg)






