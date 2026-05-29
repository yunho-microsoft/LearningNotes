# Communication Protocols

## Http vs Websocket vs socket.io

References: https://www.baeldung.com/rest-vs-websockets

Http: HTTP is a request-response based protocol. REST (Representational State Transfer) is an architectural style which puts a set of constraints on HTTP to create web services.

HTTP is a stateless protocol and works in a request-response mechanism. On every HTTP request, a TCP connection is established with the server over the socket.

The client then waits until the server responds with the resource or an error. The next request from the client repeats everything as if the previous request never happened:




Socket.IO: Socket.IO is a library that enables real-time, bidirectional and event-based communication between the browser and the server
References:
https://socket.io/docs/v4/
https://stackoverflow.com/questions/18345419/comparison-websockets-vs-socket-io

We use Server class in socket.io library. Here is the reference: https://socket.io/docs/v4/server-instance/

## HTTP and HTTPS

Request response pattern.

## Web socket

- WebSocket is a communication protocol which features bi-directional, full-duplex communication over a persistent TCP connection. Clients needs to have a stable connection with server.

    [References](https://datatracker.ietf.org/doc/html/rfc6455)
    [Websocket articles](https://www.developerfusion.com/article/143158/an-introduction-to-websockets/)

- Why need web sockets?

    We need a better way for web applications running on a client browser to communicate in real time with their servers.

### Long polling: the technique for long running connections with HTTP

[Long polling](https://ably.com/topic/long-polling) is a technique built on HTTP where the client makes a request to the server, and the server holds the connection open until new data is available or a timeout occurs. The client then reconnects to wait for more data, repeating the cycle.

- Pros:
    - Simple to implement over standard HTTP

- Cons:
    - Inefficient under load: Each client holds an open connection, which consumes server resources (threads, memory).

    - Higher latency than WebSockets: Especially when server response or timeout intervals are not finely tuned.

    - Complex client logic: Requires retry handling, exponential backoff, and timeout management.

    - Poor scalability: Difficult to horizontally scale long polling infrastructure for large user bases without significant operational overhead.

    - No guarantee of message delivery or order: Long polling doesn’t guarantee that messages will arrive in the right order - or even at all.

## SocketIo
[Reference](https://socket.io/docs/v4/)

Socket.IO is a library that enables low-latency, bidirectional and event-based communication between a client and a server.

The Socket.IO connection can be established with different low-level transports:

    - HTTP long-polling
    - WebSocket
    - WebTransport

- Socket IO guarantees message order

- [Rooms](https://socket.io/docs/v4/rooms/): A room is an arbitrary channel that sockets can join and leave. It can be used to broadcast events to a subset of clients.

### Adapter
An Adapter is a server-side component which is responsible for broadcasting events to all or a subset of clients.

When scaling to multiple Socket.IO servers, you will need to replace the default in-memory adapter by another implementation, so the events are properly routed to all clients.


#### Redis adapter in Socket IO
[Reference](https://socket.io/docs/v4/redis-adapter/)

The Redis adapter relies on the Redis Pub/Sub mechanism.

Standard Redis adapter in NodeJS has functionalites like emit to rooms but also has server level operations, like emitWithAck, server-side broadcast, fetchAllSockets.

Normal messages are implemented using Redis pattern subscribe, which subscribes to all potential channels using wildcard (*). It means one message will be sent to all SocketIo servers since they all subscribe to all channels, later the server does the filtering and sent messages to sockets with only specific room joined.

Server level operations use simple Redis subscribe to channels like request and response.

Issues with the pattern subscribe in standard Redis adapter:
- Pros:
    - No need to subscribe to new channels when a socket joins a room
    - Easier to handle multi join requests, like socket.to("room1").to("room2").emit()
    - Suitable for patterns: small number of SocketIo servers, publish messages to a subset of rooms instead of single room

- Cons:
    - Messages are replicated and sent to all SocketIO servers connecting to the same Redis adapter. It creates a lot of network traffic when the number of servers is big.
    - High server load since server is busy sending messages to all servers

FluidFramework's scenario:

    - Only need single room publishing at a time (e.g. only socket.to("room1").emit())
    - Huge number of SocketIo Servers (300+) connecting to same Redis adapter
    - Huge number of messages between clients and SocketIo Servers

FluidFramework does not fix the workflow of standard adapter and optimization has been implemented:

    - Remove pattern subscription for normal messages. Instead, change it normal subscribe when a client joins a room.
    - Keep request and response channel unchanged
    - Create default shared room subscription for server level broadcast all functionalities

It greatly reduce the number of messages in transition and the server load on Redis. It sacrifices the ability to emit to a subset of rooms at once (but still achievable if you emit to rooms one by one).
