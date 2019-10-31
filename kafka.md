# Kafka

[Source](#https://www.confluent.io/wp-content/uploads/confluent-kafka-definitive-guide-complete.pdf)

## Theory

### Why Kafka?

- Decouple producers and consumers by using a push-ull model
- Provide persistence for message data within the messaging system to allow multiple consumers
- Optimize for high throughput of messages
- Allow for horizontal scaling of the system to grow as the data streams grow

Named after *Franz Kafka*

### Terms

**Producer** : Creates messages
- Choose to receive acknowledgement for data

**Consumer** : Consumes messages

**Consumer Group** : One of more consumers that work together to consume a topic. Assures each partition is consumed by one consumer

**Topic** : A particular stream of data, similar to a able in a database

**Partition** : A split of a topic
- Each partition is ordered
- Each message within a partition gets an incremental id called offset
- Order is guaranteed within a partition

**Broker** : A kafka cluster is composed of multiple brokers (servers)
- Identified with an id
- Each broker contains certain topic partitions

**Cluster** : Brokers operate in a cluster

**Message** : Can contain a key to specify partition it should be sent to

**Ownership** : Mapping of consumer to partition

**Leader** : Broker that own a partition is called the leader

## Installing Kafka

#### Prerequisites

- Java
- Zookeeper
- Kafka

**Zookeeper** : Manages brokers
- Helps in performing leader election for partitions
- Sends notifications to Kafka in case of changes
- Kafka can't work without Zookeeper

**Ensemble** : A Zookeeper cluster

**Quorum** : A majority of ensemble members

To configure Zookeeper servers in an ensemble, they must share a common configuration file. If the hostnames of the servers in the ensmble are *zoo1.example.com*, *zoo2.example.com*, and *zoo3.example.com*, this is an example configuration file:

```
tickTime=2000
dataDir=/var/lib/zookeeper
clientPort=2181
initLimit=20
syncLimit=5
server.1=zoo1.example.com:2888:3888
server.2=zoo2.example.com:2888:3888
server.3=zoo3.example.com:2888:3888
```

Start Zookeeper

```
bin/zookeeper-server-start.sh config/zookeeper.properties
```

Start a kafka server

```
bin/kafka-server-start.sh -daemon config/server.properties
```

Create a verify and topic

```
bin/kafka-topics.sh --create --zookeeper localhost:2181
--replication-factor 1 --partitions 1 --topic test
```

Create a topic "test"

```
bin/kafka-topics.sh --zookeeper localhost:2181
--describe --topic test
```

Produce a message to test topic

```
bin/kafka-console-producer.sh --broker-list
localhost:9092 --topic test
```

Consume messages from a test topic

```
bin/kafka-console-consumer.sh --zookeeper
localhost:2181 --topic test --from-beginning
```

## Kafka Producers

1. Produce messages to Kafka by creating **ProducerRecord** which must include the topic we want to send the record to and a value. Optionally it can also specify a key and/or partition.
2. The Producer will **Serialize** the key and value objects to ByteArrays so they can be sent over the network.
3. Next the data is sent to the **Partitioner**. If we specified a partion in the **ProducerRecord**, the partitioner simply returns the partion we specified. If not, the partitioner will choose for us. It then adds that record to a batch of records that will also be sent to the same topic and partition.
4. When the broker receives the messages, it sends back a response. Success it will return **RecordMetadata** and failure it will return an error and might retry multiple times.

#### Constructing a Kafka Producer

Mandatory properties
- ```bootstrap.servers``` : List of host:port pairs of brokers that the producer will use to establish initial connection to the Kafka cluster
- ```key.serializer``` : Name of a class that will be used to serialize the keys of the records we will produce to Kafka.
- ```value.serializer``` : Name of class that will be used to serialize the values

```Java
private Properties kafkaProps = new Properties();
kafkaProps.put("bootstrap.servers", "broker1:9092,broker2:9092");
kafkaProps.put("key.serializer",
 "org.apache.kafka.common.serialization.StringSerializer");
kafkaProps.put("value.serializer",
 "org.apache.kafka.common.serialization.StringSerializer");
producer = new KafkaProducer<String, String>(kafkaProps);
```

#### Methods of Sending Messages

- **Fire-and-forget** : We send a message to the server and don't care if it arrives successfully or not.
- **Synchronous send** : We send the message, the ```send()``` method return a Future object, and we use ```get()``` to wait on the future and see if ```send()``` was successful or not.
- **Asynchronous send** : We call the ```send()``` method with a callback function, which gets triggered when it receives a response from the Kafka broker.

```Java
ProducerRecord<String, String> record =
  new ProducerRecord<>("CustomerCountry", "Precision Products",
  "France");
try {
  producer.send(record);
} catch (Exception e) {
  e.printStackTrace();
}
```

1. Create a **ProducerRecord**
2. Use the producer **send()** method to send the **ProducerRecord**. The message will be placed in a buffer and will be sent to the broker in a separate thread. Returns a **Future** object.
3. Ignore errors but may still get exceptions is the producer encounters errors

#### Sending a Message Synchronously

```Java
ProducerRecord<String, String> record =
  new ProducerRecord<>("CustomerCountry", "Precision Products", "France");
try {
  producer.send(record).get();
} catch (Exception e) {
  e.printStackTrace();
}
```

1. Here we use **Future.get()** to wait for a reply from Kafka. This will throw an exception if the record is not sent successfully to Kafka
2. If there were any errors before sending data to Kafka, print exception

**KafkaProducer** has two types of errors. *Retrievable* errors are those that can resolved by sending the message again.

#### Sending a Message Asynchronously

If we wait for a reply after each message, it will take a long time. In most cases, we don't need a reply.

In order to send messages asynchronously and still handle error scenarios, the producer supports adding a callback when sending a record.

```Java
private class DemoProducerCallback implements Callback {
  @Override
  public void onCompletion(RecordMetadata recordMetadata, Exception e) {
    if (e != null) {
     e.printStackTrace();
    }
  }
}
ProducerRecord<String, String> record =
 new ProducerRecord<>("CustomerCountry", "Biomedical Materials", "USA");
producer.send(record, new DemoProducerCallback());
```

To use callbacks, you need a class that implements **Callback** interface

#### Configuring Producers

- **acks** : Controls how many partition replicas must receive the record before the producer can consider the write successful.
  - **acks=0** : Producer will not wait for a reply
  - **acks=1** : Producer will receive a success response from the broker when the leader replica receives the message.
  - **acks=all** : Producer will receive a success response from the broker once all in-sync replicas received the message. Slowest but safest method.
- **buffer.memory** : Sets the amount of memory the producer will use to buffer messages waiting to be sent to brokers. If messages are sent by the application faster than they can be delivered, producer may run out of space.
- **compression.type** : By default, messages are sent uncompressed. This parameter can be set to *snappy*, *gzip*, or *lz4*. *Snappy* was created by Google and has good CPU performance and compression rate. *Gzip* will use more CPU time but will result in a better compression ratio.
- **retries** : Controls how many times the producer will rety sending the message before giving up.
- **batch.size** : When multiple records are sent to the same partition, the producer will batch them together. This parameter controls the amount of memory in bytes that will be used for each batch. Producer will not necessarily wait until batch is full to send it. Setting batch.size too large will cause delays.
- **linger.ms** : Controls the amount of time to wait for additional messages to be sent. Default is 0. Increases latency but also increases throughput
- **client.id** : Used for logging and metrics
- **max.in.flight.requests.per.connection** : Controls how many messages the producer will send to the server without receiving responses.
- **timeout.ms, request.timeout.ms, and metadata.fetch.timeout.ms** : These parameters control how long the producer will wait for a reply from the server when sending data.
- **max.block.ms** : Controls how long the producer will block when calling **send()**. When the limit is reached, a timeout exception is thrown.
- **receive.buffer.bytes and send.buffer.bytes** : Size of TCP send and receive buffers used by the sockets when writing and reading.

#### Ordering Guarantees

Kafka preserves the order of the messages within a partition.

If *retries* is nonzero and *max.in.flights.requests* is more than one, we cannot guarantee the order on a retry. Setting max.in.flights.requests=1 will fix this issue but will greatly decrease the throughput.

### Serialization

#### Apache Avro

Language-neutral data serialization format. Avro data is described in a language-independent schema, usually described in JSON.

#### Partitions

The **ProducerRecord** object includes a topic name, key and value. Although a key is not needed, it serves two goals; additional information stored with the message, and they are used to decide which one of the topic partitions the message will be written to. All messages with the same key go to the same partition.

If no key exists, the default partitioner is used and the record is sent to one of the available partitions at random using a round-robin algorithm.

If a key exists, Kafka will hash the key and use the result to map the message to a specific partition.

The mapping of keys to partitions is consistent only as long as the number of partitions in a topic does not change. The best solution when order matters is to have a sufficient number of partitions.



## Examples
