## Fundamentals

### Overview of overall architecture of kafka
![[Pasted image 20260616113538.png]]

- Designed as an event streaming system, so that whenever event comes in application can act immediately
- Core - Storage System - designed for storing the data efficiently as events and also designed as distributed systems
- 2 primitive apis - Producer api, consumer api
- On top of these - 2 high level api - [[#Connect API]], [[#Processing API]]
- Whole system is separated as Storage Layer and Processing Layer so that they can be scaled independently
---
## Connect API
- This is used to connect kafka with existing rest of the ecosystem
- Example, if a data needs to be imported to kafka from any source, it can be imported via source connectors (Kafka Connect & Produce Api) and if you have some data in the kafka and needs to be exported to a data sink (like search engine or graph engine) we can use kafka sink connectors to make the data flow into that sink engines
---
## Processing API
- One for java developers - Kafka Streams
- One decorative similar to SQL - ksql
---
# Event
- Event is something that happens in the world
- Each of the event is modelled in kafka as a record
- Each record has a timestamp, key, value, headers (optional)
- Value - contains payload
- Key - Enforcing ordering, colocation of data, key retention
- Both keys and values are byte arrays - why - the users can deserialize with whatever tech they want
---
# Topics
- Its a concept of organizing events of the same type
- All the events published to the topic are immutable therefore append only
---
# Partitions
- Since kafka is distributed system, we need to distribute the data within that topic to the different computing nodes in kafka cluster
- This is achieved thru concept called partitions.
- Partitions are the unit of distributing the data and parallelism
- We can configure the number of partitions per topic and for each partition the data would be typically stored within a single broker in the kafka cluster
- Although the tiered storage support (AWS S3 or GCS), we allow the data for partition to go beyond the capacity of single broker
- Each partition can be accessed independently and in parallel by writing from the producers and reading from the consumers
- Each of the event within the kafka topic partition has unique id called offset
- This offset is monotonically increasing number and once is given out it is never reused
- All the events are processed based on the offset order
---

# Planes in Kafka
1. [[#Control Plane]]
2. [[#Data Plane]]
---
## Control Plane
- Contains all the metadata of the kafka cluster
---
## Data Plane
- Contains all the actual data
- There are 2 main types of request the broker is handling:
	- Produce request from the producer
	- Fetch request from the consumer
- Both request go thru some common steps in the broker

### Producer Request handling
- Producer client starts by sending record
- The producer lib would use a pluggable partitioner that decides which partition in the topic this record should be assigned to.
- If key is specified, key is hashed by default partitioner into a particular topic partition deterministically and always route the record with same key to same partition
- If the key is not specified and record is sent in the round robin way for the next partition
- Sending each record as itself is not efficient, instead this producer library buffer all those records for particular partition in a in-memory data structure called record batches
- Accumulating the records in record batches also allow compression to be done in more efficient way
- Producer also controls when the record batch should be drained and it is controlled by 2 properties - linger.ms (by time) and batch.size (by size)
- The produced request will first land in the socker receive buffer of the broker 
- From there it would be picked up by the one of the network thread
- Once a particular thread picks up a client request from the buffer it would stick to that particular client forever (since does multiplex work across multiple clients this is only designed to do work that's lightweight)
- Network thread just picks up the bytes from the socket buffer and forms a produce request object and put that into a shared request queue
- From there the request would be picked up by the second main pool in kafka called I/O threads
- Unlike I/O threads can be used to handle request from any clients
- All of the I/O threads would be diving into the request queue to get the next request
- After the data is picked up, it validated and appended to commit log in the disk
- By default the broker would acknowledge the data only if the data is replicated across other brokers, because kafka is relied on replication and data is not flushed synchronously
- Since number of thread is limited the data to be replicated is stored in a data structure called purgatory (similar to map) and I/O threads are released for next process
- Once it completed replication the pending request would be taken out of the purgatory and will be stored in the response queue
- From there the network thread will pickup and generate a response and send to send socket buffer

### Consumer Request handling
- Similar to producer the kafka consumers send request which is consumed by the network thread and puts into the network queue
- The I/O thread processes it and puts in the purgatory