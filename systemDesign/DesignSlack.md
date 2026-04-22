# Design Slack

## Scenarios
- Message delivery when user is online
    - Multi region communication
- User presence
    - Whether a user is online(connected to server)
- Message notification when user is offline
- How user catch up missing messages

## Message delivery

- What communication protocol to use between client and server for message delivery?
Since clients need to receive messages from server in real time, we need long running connection, and WebSocket is a good candidiate here. Other options are HTTP long polling, (or something else? TBD)

- What

### How it evolves
1. Simple process:

    ClientA <-> WebSocketServer <-> ClientB

    - WebSocketServer: server that connect all clients

    - Issue: To support traffic load, we need multiple WebSocket server and we need to figure out where the message from clientA should go (which server clientB is connected). Thus we need a fanout service.

2. Add fanout

    ClientA <-> MessageFanoutService <-> WebSocketServers <-> ClientB

    - MessageFanoutService:

        service that accepts message, look up for target websocket server and send messages to the target websocket servers

    - WebSocketServer:

        Websocket servers that connect to clients for message delivery. Client may connect to different websocket servers

    - Issue:

        FanoutService need to do auth, find the websocket server and send the message. When traffic is high, messages can be jammed. When the service crashes before message is sent, the message is lost. We need a queue services to temporarily store the messages. Kafka is a good choice

3. Add Kafka

    ClientA <-> MessageService <-> Kafka <-> FanoutService <-> MultipleWebSocketServers <-> ClientB

    - Kafka: MessageService will simply push messages to Kafka Queues. FanoutService will consume messages from KafkaQueues, process it by finding out which websocket server to route and send the messages to the websocket servers

### Communication protocols
1. How client receive messages?

    Through websocket connecitons to receive messages in real time

1. How client send messages?
    - Option1: through HTTP request
        - Natural way to receive the response about whether a message is sent correctly or not
        - Client needs to keep 2 connections: websocket for message receiving and HTTP for message sending
        - Make websocket server stay out of the message sending flow. They can be dump pipes

    - Option2: through the same websocket connection
        - Only keep one connection between client and server
        - But need a separate message design to make sure client receives the response about whether a message is sent or not. Fire and forget is not a good approach here.
        - Websocket servers will be involved in message sending path. And they are stateful now since they need to store the message and tell clients whether a message is sent or not.

1. The communication between microservices
    - Option1: gRPC
        - gRPC uses HTTP/2 — multiplexed streams on one persistent connection, no handshake per call. Protobuf encoding ~3× smaller than JSON. Strongly typed contract. Best for high-frequency internal service calls.

    - Option2: REST calls (HTTP)
        - Human readable request content since request payload is JSON encoded but has larger request size
        - Request/response are loosely coupled. No format definition needed before hand.
        - REST is helpful when no control on client and servers since it needs a .proto file to define the request/response format

### Message flow for single region

1. ClientA logs in with authentication service

1. PresenceService --update client presence--> Redis

1. ClientA --send messages--> Message service

1. Message --send message--> Kafka

1. FanoutService <--consume message-- Kafka

1. FanoutService --Find all present clients(and corresponding websocket servers)--> PresenceService

1. FanoutService --send messages--> WebsocketServers

1. Each websocket server --send messages--> Client B,C, D

### How to scale it globally
If all services are located on same datacenter in one area, users from different areas need to do cross-region request and latency is high. So we need to scale it globally.

We need to have same set of services for each region.

Now we have a few issues:

#### How does service in US knows a target client is present in Asia?

We need to have a shared, global presence service. A service that tracks all clients' presence across the globe and allow services from all regions to query who is present. The global registry needs to be a **globally distributed, low-latency read** store.

This global registry only store simple user presence data. Something like:
```json
{
    "user_id": "Bob",
    "status": "Online",
    "active_region": "Asia",
    "last_updated": "1711800000"
}
```

#### How does Bob receive his message in Asia when Alice in US messages him?

- Option1: Bob connects to Asia websocket servers

    Good thing about this approach is that if Bob send/receive messages from another user in Asia, the latency is optimal.

    In this case, we need to forward the message across regions.

    - option 1-a: Have a regional API gateway that accepts request from other regions. Once the fanout service in US region finds that Bob is active in a different region, it sends the request to the API gateway in Asia with the message, and Asia API gateway will push the message to regional Kafka queue for fanout services in Asia to send it to Bob.

        - Pros: the gateway service in each region can potential batch messages and reduce cross region calls.

    - option 1-b: Replicate message across regions. Each message in each regional Kafka is replicated to other region's Kafka queues. Fanout service just drop the message if the receiver is not active in its region.

        - Cons: Requires huge traffic between regions since each message needs to be replicated.

Option2: Bob always connect to his home region

    Pros: simple design but long latency between all messages if Bob is not in his home region.

    This option is valid if the app has data residency requirements, for example, EU requires the processing and storage of data in EU regions. In this case, clients from EU need to have their data stored in EU region only. For example, Slack has the concept of workspace and each workspace has a region. Users can only talk to others belonging to the same workspace.

## Message retrival for missing messages
Message id should be sortable and increasing by timestamp. It is for easy sorting by timestamp. Check Snowflake Id for example.

Message id may or may not be consecutive since it is hard to make it consequtive in a distributed system.

The App should keep 2 message ids:
    - last_received_message_id: the last message id a client received, store on client side
    - last_read_message_id: the last message id that a user read, stored on server

### How does client retrieve missing message when server fail to send it?

- If message id is consecutive, then it is easy for clients to recover since it can quickly find the missing range (received 1,2,3, expecting 4 but got 5)

- If message id is only in ascending order but not consecutive, then need other mechanisms

## Message Notification when user is offline

## Presence service design: whether a user is online


