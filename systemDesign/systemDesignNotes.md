# System design

# Database and storage

- Query optimization / Denormalization / Partition/ Sharding
- N+1 Query
- Primary index vs secondary index vs global index

- How to prevent race condition
- How to prevent transaction overwrite (locking / lock free approaches)
- ACID / row lock / distributed lock

- MySQL vs NoSQL vs TimeSeries DB

- Consistent hashing

Storage?
- OLAP vs OLTP: OLTP (Online Transaction Processing) and OLAP (Online Analytical Processing)
- ELK: Elastic stack?

## ACID
- Atomicity: All operations in a transaction succeed or all fail — no partial writes.

How to achive: write ahead log?

- Consistency: A transaction moves the database from one valid state to another. Constraints (CHECK, UNIQUE, FK, NOT NULL) are enforced at commit time.

- Isolation: Concurrent transactions behave as if they're running one at a time.

How to achieve: locks or multi-version concurrency control

- Durability: Once a transaction is committed, the data survives power loss or crash.

## How DB handles concurrency?

### SQL define a few scenarios
- SQL defined a few isolation levels: Read uncommitted, Read committed, Repeatable read and Serializable, from weak to strong consistency

- SQL defines a few scenarios of whether they are allowed under above levels:
    - dirty read: A transaction reads data written by a concurrent uncommitted transaction.

    - nonrepeatable read: A transaction re-reads data it has previously read and finds that data has been modified by another transaction (that committed since the initial read).

    - phantom read: A transaction re-executes a query returning a set of rows that satisfy a search condition and finds that the set of rows satisfying the condition has changed due to another recently-committed transaction.

    - serialization anomaly: The result of successfully committing a group of transactions is inconsistent with all possible orderings of running those transactions one at a time.


### Multi version concurrency control vs Locks

- Multi version concurrency control
A update to a row only add a newer version (snapshot) rather than updating the existing one. A read will read the snapshot earlier than its execution. It boosts read transactions a lot. Write lock is still needed when conconcurrent writes happen on the same row.

    - Pros: read and write does not interfere with each other and can support high read throughput

    - Cons: complex logic to manage versions. Table size can increase a lot if cleanup does not happen in time, since old snapshot lingers around



- Use Locks
Use locks on table and rows for read and writes. Shared locks can be used for concurrent reads and exclusive lock should be used for writes until it commits
    - Pros: Easier in concepts. No need to clean up old versions

    - Cons:
        - Readers block writers and writers block readers, which reduces concurrency.
        - Long-running reads hold shared locks that prevent any updates to those rows.
        - Deadlocks are a real concern — two transactions can each hold a lock the other needs, requiring the database to detect and abort one.
        - Throughput tends to suffer under mixed read/write workloads.

### Normalization and DeNormalization

- Normalization: put data into different tables. Example: you have users, orders and produtcts table to store user, order and product information and use userId, orderId and productId in order instead of copying all data to the order table
    - Pros: one update and it updates everywhere else
    - Cons: need to JOIN tables to get full information

- Denormalization: replicate some data in the tables, like include username in the order table
    - Pros: avoid complex JOIN in the table
    - Cons: to update a record, you need to update multiple table potentially since data is replicated everywhere

## PostgreSQL

- What is it?
Open sourced SQL DB

- When to use it?

- How does it work?

## ElasticSearch
- What is it?
A DB optimized for searching top relevant results, based on Apache Lucene project

- When it should be used?
Complex searches like find a keyword in multiple field and show the top relevant results

- How does it work?

# Backend and efficiency

- Speed up on read path
- speed up on write path
- caching strategies
- cache warm up
- sharing: push vs pull

- fanout vs hotspot (celebrity effects)

High availablity:
- Usage too high issues (connection count, CPU, memory, disk, storage, cache)
- CP vs AP (CAP theory)

Pagination (cursor vs offset)

# API design
- API gateway

- talk to 3rd party API (backoff, retry, timeout, circuit breaker, random select, local fallback)

## RESTful - HTTP with GET, POST, PUT, UPDATE, DELETE
- self contained

## GraphQL

- What is it?
GraphQL is a query language and runtime for APIs, developed by Facebook in 2012 and open-sourced in 2015.GraphQL is a query language and runtime for APIs, developed by Facebook in 2012 and open-sourced in 2015. It gives clients the power to ask for exactly the data they need — nothing more, nothing less — in a single request.

Instead of hitting multiple endpoints with fixed data shapes (as in REST), you send a query describing the structure of the data you want, and the server returns JSON matching that exact shape.

## gRPC
- References:
https://grpc.io/docs/what-is-grpc/introduction/

- Check systemDesign/gRPC.md for details

1. gRPC uses ProtocolBuf as message payload (not JSON) and HTTP/2 (not HTTP/1.1) for communication. So smaller message size

1. gRPC is faster than REST since it is using HTTP/2, which allows many concurrent streams over a single TCP connection, no blocking. REST is using HTTP/1.1 and It is one request at a time per connection — causes head-of-line blocking

1. Mainly used for communication between microservices with high traffic: 1000+ request per sec to reduce overhead

1. REST is still widely used in public APIs and modern browsers since gRPC has its limitions: need to recompile .proto file for client when changes happen.

1. gRPC is more suitable when you have control over both client and server side

## Long term connection

### Server-Sent Events (SSE)

- a standard built on top of a single long-lived HTTP connection. The server pushes text-based messages to the client over time using the text/event-stream content type.

- Unidirectional. Only server to clients

- The SSE connection cannot be opened for too long and defines EventSource object to make it automatically reconnect with the ID of the last message received. Servers are expected to keep track of prior messages that may have been missed while the client was disconnected and resend them.

### Websocket

WebSockets provide a persistent, TCP-style connection between client and server, allowing for real-time, bidirectional communication with broad support (including browsers).

- Bidirectional and persistent TCP connection between clients and server
- Broad support, including browsers
- Websocket does a upgrade from HTTP whenever it is OK to do so.

### HTTP long polling

HTTP Long Polling is the simplest and oldest technique. The client sends a regular HTTP request, but instead of responding immediately, the server holds the connection open until it has new data (or a timeout occurs). Once the client receives a response, it immediately sends another request, creating a loop. It works everywhere HTTP works, which is its main advantage, but it's inefficient — each cycle involves full HTTP headers, and the server has to manage many hanging connections. It's largely a legacy approach now.

# Messaging system
- Kafka vs AWS SQS vs Redis + celery vs RabbitMQ

- stream processing (Flink)
- Batch processing (spark)

- Event loop (python and nodejs's event loop?)


## Question to answer:

1. How does it make sure messages are not lost?
At least once or at most once?

2. How does it distribute messages?
Multiple worker consume same queue to distribute the load?
Fan out messages: each consumer receive replica of messages?

## Kafka
Reference: https://kafka.apache.org/documentation/

Kafka does not have retry logic

Questions:

1. How Kafka handle offset?
Manage message position consumed in a partition

2. Kafka is disk based? How does it make it fast?

3. Zookeeper used in Kafka?

4. Pull or push model?

5. Livesite issues encountered with Kafka
Hard to manage SSL certificates in master and worker nodes



## RabbitMQ

## Redis Stream
- Reference: https://redis.io/docs/latest/develop/data-types/streams/

- Each entry (or message) is identified by an ID that is generated by Redis by default. The [entry id](https://redis.io/docs/latest/develop/data-types/streams/#entry-ids) format is <millisecondsTime>-<sequenceNumber>

Adding an entry to a stream is O(1). Accessing any single entry is O(n), where n is the length of the ID. Since stream IDs are typically short and of a fixed length, this effectively reduces to a constant time lookup. For details on why, note that streams are implemented as radix trees.


### Redis stream support 3 types of [queries](https://redis.io/docs/latest/develop/data-types/streams/#getting-data-from-streams): querying by range, listening for new items, consumer groups
- Query by range:
XRANGE complexity is O(log(N)) to seek, and then O(M) to return M elements

- Listen for new item: XREAD, it replicates messages to all consumers (fan-out)
A stream can have multiple clients (consumers) waiting for data. Every new item, by default, will be delivered to every consumer that is waiting for data in a given stream. This behavior is different than blocking lists, where each consumer will get a different element. However, the ability to fan out to multiple consumers is similar to Pub/Sub.

- Consumer group: it is completely different from the concept of Kafka consumer group.

    In a single Redis stream, multi coonsumers in a consumer group can receive message out of order. To achieve Kafka partitions, you have to use multiple keys and some sharding system such as Redis Cluster or some other application-specific sharding system.

    Basically Kafka partitions are more similar to using N different Redis keys, while Redis consumer groups are a server-side load balancing system of messages from a given stream to N different consumers.

# Database

## SQL
SQL supports search by any columns without the need of creating indexes but indexes help the performance

## NoSQL
NoSQL doesn't support search by any attributes?

### NoSQL single table design
- Amazon Dynamo DB single table design:
Reference:
    - https://aws.amazon.com/blogs/compute/creating-a-single-table-design-with-amazon-dynamodb/
    - https://dynobase.dev/dynamodb-joins/


- Mongo DB
    - [Data duplication](https://medium.com/mongodb/data-duplication-in-mongodb-and-modern-applications-5a5c1836d114)
    - [The Single Collection Pattern](https://www.mongodb.com/developer/products/mongodb/single-collection-pattern/)

### Amazon Dynamo DB
- Core components: https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html
- Issue with Dynamo DB: keys and indexes have to be designed based on query pattern, like how you are going to query the data

# Load balancing
Load balancing can be done at client side or server side

## Client side load balancing
The idea is the let client first send request to get a list of available servers and pick the best one (lowest latency for example).

Some examples of client side load balancing:

    - Redis cluster: TBD
    - DNS lookup: DNS resolver returns a list of rotated servers when you look up DNS address


- Pros: Fast on client side and less burden on server side.

- Cons:
    - Not able to update the server list instantly since each client may cache the list of servers
    - Clients will have knowledge of the list of servers available

## Server side load balancing
Server side load balancing requries a load balancer.

Load balancer can be categorized into L4 loadbalancing and L7 load balancing.

Used when we want to quick update of server list or we do not want clients know the list of servers available

### L4 level load balancer
Operates at transport layer and make routing decisions based on network information like IP addresses and ports, without looking at the actual content of the packets.

- Works as if client directly connects to that routed server
- Maintain persistent TCP connections between client and server.
- Fast and efficient due to minimal packet inspection.
- Cannot make routing decisions based on application data.

### L7 level load balancer
Operates at application level.It receives an application-layer request (like an HTTP GET) and forward that request to the appropriate backend server. It keeps separate connections: connections between clients and LB, connections between LB and backend servers.

- LB is in between clients and targeted backend servers.
- It can do complex routing by checking request headers, URL, cookies.
- Can route based on request content (URL, headers, cookies, etc.).
- More CPU-intensive on LB and slight latency due to packet inspection.

When to use it:
    - More suitable for RESTful requests
    - May not suitable for long lasting connections, like websocket, if the concurrent connections is huge.

## Combinations of load balancing

### Example: HTTP and websocket traffic on a huge number of long lasting concurrent clients
If same client's requests (same IP) needs to be handled by separate servers, then L4 LB is not a good option.

Assume L7 LB cannot support that many of connections at the same time, we can use combination of client load balancing + L7 load balancing.

Clients can request endpoints information on L7 LB endpoints. After getting the endpoints, client directly connects to the backend server endpoints. This is done by Azure Fluid Relay Server to handle huge number of concurrent websocket connections (which L7 LB cannot handle)

# Handle failures
- APIs should be designed as idempotent if possible, meaning they can be safely retried
- Retries can help mitigate transient issues
- Backoff (increases the time between subsequent retries, which keeps the load on the backend even.) can reduce the burden on the system if server is already at high load.
    - Exponential backoff, where the wait time is increased exponentially after every attempt. With better with a cap to avoid unlimited long time of retries
- Jitter can help the situation where retries is ineffective when all clients retry at the same time. This is a random amount of time before making or retrying a request to help prevent large bursts by spreading out the arrival rate.



# CDNs

# File upload and distribution
- Client upload vs Server upload
- pre-signed URL / Multipart upload on client

# Communication protocols:
- cookie vs local storage

# Python concurrency
- Process vs thread vs coroutine
- python when to use multiprocessing / threading / asyncio

- ASGI vs WSGI

# Serialization
- Serialization and deserialization
- Bloom Filter