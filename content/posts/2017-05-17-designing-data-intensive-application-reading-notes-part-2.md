---
title: Designing data intensive application, reading notes, Part 2
author: will
date: 2017-05-17T09:12:44+00:00
url: /2017/05/17/designing-data-intensive-application-reading-notes-part-2/
categories:
  - tech
tags:
  - lang-en
  - book

---
Chapter 4, 5, 6

## Encoding formats

xml, json, msgpack are text based encoding format, they can&#8217;t carry binary bytes (useless you encode them in base64, size grows 33%). And they cary schema definition with data, wast a lot of space.

thrift, protobuf are binary format, can take binary bytes, only carry data, the schema is defined with IDL(interface definition language). They have code generation tool to generate code to encode and decode data, along with check. Every field of data is binded with a tag(mapped to a field in IDL file). If a field is defined is required, it can&#8217;t by removed or change tag value, otherwise old code will not be able to decode it.

avro (used in hadoop), have a write schema and a read schema, when store a large file in avro format(contain many records with same schema), the avro write schama file is appended to the data. If use avro in RPC, the avro schema is exchanged during connection setup. When decoding avro, the lib will look both write schema and read schema, and translate write schema into read schema. Forward compatibility means that you can have a new version of the schema as writer and an old version of the schema as reader, backward compatibility means that you can have a new version of the schema as reader and an old version as writer.

## Replication

purpose:

  * reduce latency (replication via geo location)
  * HA
  * increase read throughput

### sinle-leader replication

Also called master/slave or active/passive

Use case: postgresql, MySQL, Oracle Data guard, SQLServer Always On, MongoDB, RethinkDB,Espresso, Kafka, RabbitMQ, DRBD&#8230;.

If replication is synchronous, leader is usually setup to replicate to one slave synchronously, to other slaves asynchronously.

Complete asynchronously replication is used more often.

Process of setting up new slave:

  * Take snapshot of leader&#8217;s database, without lock if possible. (like innobackupex for MySQL)
  * Copy snapshot to slave.
  * Slave connects to the leader and requests all the data changes that have happened since the snapshot was taken (snapshot must have position info)
  * slave wait until catch up with master, then it can handle incoming data. 

Failover for leader:

  * Determining leader has failed. Failure may have many reasons, usually use timeout.
  * Choosing a new leader. Choose slave with least lag with leader, let all slave agree on new leader involved consensus problem(raft, paxos)
  * Reconfigure system to use new leader.

## multi-leader

Also called master/master or active/active. Usually used in multi data centers, every data center have a master.

Benefit:

  * Can make database closer to users in geo, reduce latency.
  * If one master fails, promote remote master to be new master.

related projects:

  * Tungsten Replicator for MySQL 
  * BDR for PostgreSQL
  * GoldenGate for Oracle

Some db features will be problem when working with multi-leader: autoincreasementing keys, triggers,integrity constraints.

multi-leader replication must resolve data conflict problem:

  * try to avoid conflict completely, if one user&#8217;s action is restricted to one leader strictly(maybe based on location), you won&#8217;t come to this issue.

### leaderless

System like amazon&#8217;s dynamo(raik, openstack swift).

Write requests are sent to multi replicas at the same time, client needn&#8217;t wait for all replicas response ok, use formula `W + R > N`(N is # of configured replicas) to decide whether the write successed or not. You can configure the value of W and R to make system more efficient for write or read requests.

A backgroup repair will check data consistent between replicas and resolve conflicts, so value should have a version.

## Partition

Based on key range or hash.

Hash can reduce hot spots than key range (since hash value is meanless), but if a key is very hot, may still come to problem. A protential solution is adding a random number(2 digits) at the begin or end of the key to split the key across 100 keys, but whening reading, you need to read whole 100 keys and combine them (not very clear, if need to write 100 times and read 100 times, what&#8217;s the meaning?).

### Rebalancing

If you use hash, better to devide possible hash value into ranges, and assign each range a partition. But you can&#8217;t simply mod # of nodes. Since if add nodes, the value will change, and you must rebalancing all your data.

Better way is to do presharding, create much more partitions than nodes. You can mod # of partitions to place your keys, then place the partitions to ndoes. If you need to add more nodes, just move partitions, key -> partition mapping will not change. Raik, ElasticSearch, Couchbase, Voldemort and RedisCluster use this way.

### Dynamic partition

For key range based partition, if you configure the range wrong, the distribution will be very skewed, reconfigure the range is very tedious.

HBase and RethinkDB create partitions dynamically. If a partition grows to specific size, it will be splited into two partitions.If lots of data are deleted, it can be merged with a adjacent partition.

### Request routing

One way is use service discover(mostly, rely on zookeeper) to track partition assignment. When partitions changed, ZooKeeper will notify its client where to find the moved partition.

Another way is use gossip protocol to tell all nodes the change in cluster.Request can be sent to any nodes, then the node will forward the request to appropriate node for the requested partition. (Cassandra, Raik, RedisCluster, Consul)
