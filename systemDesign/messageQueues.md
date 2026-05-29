# Messaging system

## Why message queue?

- Decoupling & Asynchronous Processing: Producers and consumers (different parts of an application) operate independently, improving flexibility. A service sends a message and moves on, rather than waiting for processing, boosting system responsiveness.

- Reliability & Fault Tolerance: Messages are durable and stored until processed, preventing data loss if a service crashes. Queues handle retries and ensure delivery.

- Scalability & Load Balancing: Easily scale by adding more consumers to process messages from the queue, distributing work and handling traffic surges effectively.

- Throttling & Buffering: Act as a buffer, smoothing out peaks in demand, preventing consumers from being overwhelmed by sudden spikes in requests.

- Background Processing: Offload time-consuming tasks (like video transcoding, report generation, or sending bulk emails) to background workers, keeping user-facing applications fast.

- Clearer Architecture: Define clear boundaries between microservices, making systems easier to develop, test, and maintain. 

## Key points

- Batch process vs stream process

    Batch process: process a very large volume of data in a single workload

    Stream process: process small units continuously in real-time flow

- Stream process framework: Kafka vs AWS SQS vs Redis+Celery vs Redis Stream vs RabbitMQ

- stream processing (Flink)
- Batch processing (spark)

- Event loop (python and nodejs's event loop?)

- Consuming message: Push vs Pull
    - Push: the server sends messages to subscribers once it is available
    - Pull: the client pull messages from server. Http long polling can be used here to improve performance.


## Questions to answer about a message queue:

1. How does it make sure messages are not lost?

    Most queue services have at least once guarantee (with some ack mechanism implemented).

1. At Least Once or At Most Once or Exactly Once?
    
    Usually queue services guarantees at least once to make sure messages are not lost. In reality, strict exactly once delivery is impossible, but it can be achieved with some nuances, which is called Effectively-Once (for example, the consuming logic has some deduplicate logic or is idempotent. Then the entire pipeline can be thought as effectively once).

    [References](https://nryanov.com/overview/messaging/delivery-semantics/processing-semantics/exactly-once/at-least-once/at-most-once/delivery-and-processing-semantics/#delivery-semantics-overview-)

1. How does it make sure messages are consumed in order?
It usually requires queue services to have a single consumer to make sure all messages are delivered in order.
If multiple consumers lisen for the same queue, it is possible that some messages are processed earlier due to consumer issues.

    All queue services have similar features

    | Kafka | Redis stream | RabbitMQ |
    | - | -| - |
    | Each partition in a consumer group can only have one consumer and it guarantees that messages consumed in order within a partition | A single consumer on the queue guarantee the messages delivered in order | A single consumer on the queue guarantee the messages delivered in order |


1. How does it distribute messages?

    Multiple worker consume same queue to distribute the load?
    
    Fan out messages: each consumer receive replica of messages?

1. How to limit the queue size?

    | Kafka | Redis stream | RabbitMQ |
    | - | -| - |
    | Kafka has data retiontion policy, either by time (retiontion.hours) or by size (retiontion.bytes). Older messages are removed if a message is older than the retention duration or the topic's segment is larger than the limit | User need to explicitly trim or delete messages from Redis stream | RabbitMQ queue size can be set by user. A message is marked for deletion once acknowledged by consumer. |

## Kafka

### References
[Official documentation](https://kafka.apache.org/41/getting-started/introduction/)

### Questions:
1. How Kafka handle offset?
    
    [Link](#kafka-consumer-offset)

2. Kafka is disk based? How does it make it fast?

    [Link](#kafka-is-disk-based---link)

3. Zookeeper used in Kafka?

    [Link](#zookeeper-in-kafka)

4. Pull or push model?

    [Link](#push-vs-pull-model)

5. Livesite issues encountered with Kafka clusters
    - Hard to manage SSL certificates in master and worker nodes. Need a fully managed cluster
    - Need to have separate Zookeeper clusters for older version Kafka clusters
    - No built-in dead letter queue (DLQ).

### Concepts:
- **Event**: somethign happened. An event has a key, value, timestamp, and optional metadata. Kafka is a event queue

- **Producer** and **Consumer**: Producers are those client applications that publish (write) events to Kafka, and consumers are those that subscribe to (read and process) these events.

- **Topics**: Events are stored in topics. Topics are like folders and events are files in the folder. Events in a topic can be read as often as needed—unlike traditional messaging systems, events are not deleted after consumption

- **Broker**: Some of these servers from the storage layer, called the brokers. (Kafka server side)

- **Partition**: Topics are partitioned, meaning a topic is spread over a number of "buckets" located on different Kafka brokers. Events with the same event key (e.g., a customer or vehicle ID) are written to the same partition, and Kafka guarantees that any consumer of a given topic-partition will always read that partition's events in exactly the same order as they were written.

    **Event order is only guaranteed in partition level.**

- **consumer group**: In Kafka, a consumer group is a set of consumers from the same application that work together to consume and process messages from one or more topics. Remember that each topic is divided into a set of ordered partitions. Each partition is consumed by **exactly one** consumer within each consumer group at any given time.

### Kafka is disk based - [link](https://docs.confluent.io/kafka/design/file-system-constant-time.html)
Disk read/write are much faster for linear read/write, comparing to random read/write. Instead of keeping data in memory and flush to disk when running out of space, Kafka immediately write data to a persistent log on the filesystem without necessarily flushing to disk. In effect this just means that it is transferred into the kernel's pagecache.

### Kafka consumer offset:
References:
https://kafka.apache.org/documentation/#theconsumer
https://kafka.apache.org/22/javadoc/org/apache/kafka/clients/consumer/KafkaConsumer.html

Consumer needs to commit the offset (committed position), which will be used to tell where the consumer should recover to once it is getting restarted. The consumer application need not use Kafka's built-in offset storage, it can store offsets in a store of its own choosing.

Kafka does not have explicit retry logic: when a consumer fails to process a message, it does not commit the offset within the timeout. The message is not considered consumed and will be delivered to next consumer when it reconnects to the partition

### Kafka session.timeout.ms vs max.poll.interval.ms

- session.timeout.ms:

    The timeout used to detect client failures when using Kafka’s group management facility. The client sends periodic heartbeats to indicate its liveness to the broker. If no heartbeats are received by the broker before the expiration of this session timeout, then the broker will remove this client from the group and initiate a rebalance.

- max.poll.interval.ms

    The maximum delay between invocations of poll() when using consumer group management. This places an upper bound on the amount of time that the consumer can be idle before fetching more records. If poll() is not called before expiration of this timeout, then the consumer is considered failed and the group will rebalance in order to reassign the partitions to another member.

- Differences:

    session.timeout.ms controls the heartbeat and max.poll.interval.ms controls the maximum time a task should take to process. This separation makes it possible to set a longer process time without affecting the heartbeat interval (detect earlier failures when processing time is long). But expiration of either period will trigger a rebalance.

### Push vs pull model
Kafka queue use pull stratgy (consumer pull messages from server), reference: https://docs.confluent.io/kafka/design/consumer-design.html#push-versus-pull-design



### Kafka partition rebalancing
Rebalancing comes into play in Kafka when consumers join or leave a consumer group. In either case, there is a different number of consumers over which to distribute the partitions from the topic(s), and, so, they must be redistributed and rebalanced.

[Useful video](https://www.youtube.com/watch?v=ovdSOIXSyzI&t=160s)

- Kafka group leader (one of the consumer in the consumer group) is elected by Kafka group coordinator (one Kafka broker)
- Kafka group leader is responsible for calculating the partition assignment, rather than the broker (group coordinator). It makes the partition assignment pluggable (customizable by user, since some consumer may be more powerful than others)

#### Kafka rebalancing logic

[Reference](https://www.confluent.io/blog/debug-apache-kafka-pt-3/#:~:text=Rebalancing%20comes%20into%20play%20in,must%20be%20redistributed%20and%20rebalanced)

- Stop the world rebalance

    It is like every consumer should stop the work (being revoked the partition), send join group request, and only start work when the partition assignment is complete.

    - Issues:
        - consumers need to wrap up their work and if they are assigned to the same partition as before, then the wrap up work is wasted.
        - Work has been completely stoped during the rebalancing phrase. Sometimes the rebalancing takes longer when all consumers are restarted in scenarios like an update. In this case, multiple join group request can be sent and takes longer to complete the rebalancing


- Incremental Cooperative rebalance
    
    Instead of revoking all partition assignment for all consumers, they continue to work. And only the changed partition assignment is sent back to consumers. If consumers have same partition assignment as before, the process is still going.

- Static group membership (avoid rebalancing)

    Assign a static group member id to each consumer in the group. If a consumer is restarted, the rebalance won't happen unless the session is timed out.


### Zookeeper in Kafka
Zookeepr is used to manage metadata in Kafka.

ZooKeeper provides several fundamental services for distributed systems, including primary server election, group membership management, configuration information storage, naming, and synchronization at scale. Its primary aim is to make distributed systems like Kafka more straightforward to operate by providing improved and reliable change propagation between replicas in the system. [references](https://github.com/AutoMQ/automq/wiki/What-is-the-Zookeeper-in-Kafka-All-You-Need-to-Know)

Zoopkeeper is not involved in the message processing part in the broker (more like a control plane).

Kafka community is moving to KRaft from Zookeeper. Starting from Kafka version 4.0, Zookeeper is removed and KRaft is the exclusive protocol.

- Zookeeper vs KRaft performance comparison

    [Partitions and Data Performance, part 1](https://www.instaclustr.com/blog/apache-kafka-kraft-abandons-the-zookeeper-part-1-partitions-and-data-performance/)
    
    [Partitions and Data Performance, part 2](https://www.instaclustr.com/blog/apache-kafka-kraft-abandons-the-zookeeper-part-2-partitions-and-meta-data-performance/)

### When should use Kafka:

Use Kafka when:
    - Need to replay events
    - Streaming events in high throughput: large amount of real time messages 
    - Process events in order (order is guaranteed within a partition)


Do NOT use Kafka when:
    - Long running tasks: Kafka relies on consumer to commit offset and not retry logic on failed messages.
    - Complex routing mechanism
    - Simply pub sub: Kafka is heavy and overkill for simple pub sub scenarios
    - Low latency (<10 ms>). Kafka P99 latency is around 100-200ms in a cloud environment


## RabbitMQ

### Message acknowledgement
[reference](https://www.rabbitmq.com/tutorials/tutorial-two-go#:~:text=Message%20acknowledgment%E2%80%8B,if%20the%20workers%20occasionally%20die.)

## Redis Stream
- Reference: https://redis.io/docs/latest/develop/data-types/streams/

- Each entry (or message) is identified by an ID that is generated by Redis by default. The [entry id](https://redis.io/docs/latest/develop/data-types/streams/#entry-ids) format is <millisecondsTime>-<sequenceNumber>

- Adding an entry to a stream is O(1). Accessing any single entry is O(n), where n is the length of the ID. Since stream IDs are typically short and of a fixed length, this effectively reduces to a constant time lookup. For details on why, note that streams are implemented as **radix trees**.


### Redis stream support 3 types of [queries](https://redis.io/docs/latest/develop/data-types/streams/#getting-data-from-streams): querying by range, listening for new items, consumer groups
- Query by range:
XRANGE complexity is O(log(N)) to seek, and then O(M) to return M elements

- Listen for new item: XREAD, it replicates messages to all consumers (fan-out)
A stream can have multiple clients (consumers) waiting for data. Every new item, by default, will be delivered to every consumer that is waiting for data in a given stream. This behavior is different than blocking lists, where each consumer will get a different element. However, the ability to fan out to multiple consumers is similar to Pub/Sub.

- Consumer group: it is completely different from the concept of Kafka consumer group.

    In a single Redis stream, multi coonsumers in a consumer group can receive message out of order. To achieve Kafka partitions, you have to use multiple keys and some sharding system such as Redis Cluster or some other application-specific sharding system.

    Basically Kafka partitions are more similar to using N different Redis keys, while Redis consumer groups are a server-side load balancing system of messages from a given stream to N different consumers.


## Spark

TBD

Is it data processing framework rather than temporary message storage and transporting?

Spark was created to address the limitations to MapReduce, by doing processing in-memory, reducing the number of steps in a job, and by reusing data across multiple parallel operations. [reference](https://aws.amazon.com/what-is/apache-spark/)

## Flink

TBD