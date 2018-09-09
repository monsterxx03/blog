---
title: ElasticSearch cluster
author: will
date: 2017-03-22T16:22:32+00:00
url: /2017/03/22/elasticsearch-cluster/
categories:
  - tech
tags:
  - distribute
  - elasticsearch

---
In this article, let&#8217;s talk about ElasticSearch&#8217;s cluster mode, which means multi nodes ElasticSearch.

## Basic concepts

cluster: A collection of server nodes with same `cluster.name` settings in elasticsearch.yaml

primary shards: Divide a index into multi parts(by default 5), shards of an index can be distributed over multi nodes. It enables scale index horizontally and make access to index parallelly(accross multi nodes).

replicas: backup for shards, also replicas can handle search requests, which means you can scale your search capacity horizontally via replicas.

<!--more-->

## Single node

If you setup a single node ES, all shards will be hosted together, and there will be no replicas.

Host replicas and shards on a same node make no sense, they will be lost at same time. If no additional node is available, ES will not allocate replicas.

Get all shards info: `curl http://localhost:9200/_cat/shards?v`

    forum                 2     p      STARTED       0  159b 127.0.0.1 dev-host
    forum                 2     r      UNASSIGNED
    forum                 3     p      STARTED       0  159b 127.0.0.1 dev-host
    forum                 3     r      UNASSIGNED
    forum                 4     p      STARTED       0  159b 127.0.0.1 dev-host
    forum                 4     r      UNASSIGNED
    forum                 1     p      STARTED       0  159b 127.0.0.1 dev-host
    forum                 1     r      UNASSIGNED
    forum                 0     p      STARTED       0  160b 127.0.0.1 dev-host
    forum                 0     r      UNASSIGNED
    

You can see, by default, a index will have 5 primary shards(symbol `p`), all replicas(symbol `r`) are not assigned.

Also cluster status will be yellow, since there&#8217;re no replicas available.

## Two nodes

Let&#8217;s add one node into cluster.

ES have two ways to find other nodes in cluster: multicast and unicast.

`multicast` works via send a UDP ping to local network, in production environment, we should disable it(disabled by default), I think no body will like it.

`unicast` works via provide an initial host list (defined by `discovery.zen.ping.unicast.hosts`), nodes communite with each other via gossip protocol. So a new node needn&#8217;t know complete list of cluster, it can only talk to a few nodes to get whole cluster info. Some projects like consul and redis cluster also work like this way.

One more thing to care is you need to specify `network.publish_host` explicitly. By default it will be 127.0.0.1, you need to change it to a ip/host which other nodes can access.

![two-nodes-es][1]

In two nodes setup, all replicas will be hosted on another host. All write operations to NODE 2 will be forwarded to NODE 1 internally on server. As comparison, redis cluster use so called smart client, server only tell client the right one to query.

With two nodes setup, we can suffer one node down with no data loss. However, network partition will make two nodes brain split.

Each node think another one is down, so it make self be master. At the same time, client&#8217;s write operations still sucess on both. But data will split on two nodes, you will not be aware of it unless search return wired result. So you may come to data inconsistency on two nodes setup.

Cluster status will be green, since all shards will have one replicas.

### Three nodes

Three nodes can suffer network partition. But you need to set `discovery.zen.minimum_master_nodes` to 2. If one node failed, left two will elect a new master. If two nodes failed, there will not be enough master eligible nodes, client will come to error, so no data inconsistency will happen.

![three-nodes-es][2]

Picture taken from es official doc, it uses 3 replicas as example.

### Notes

ES has very flexible configuration, you can setup dedicated master eligible node, data only node, ingest only node, tribe noly node based on your size: https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html

 [1]: http://blog.monsterxx03.com/wp-content/uploads/2017/05/two-nodes.png
 [2]: https://blog.monsterxx03.com/wp-content/uploads/2017/05/three-nodes.png
