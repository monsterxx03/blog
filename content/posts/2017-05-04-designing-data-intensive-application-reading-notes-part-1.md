---
title: Designing data intensive application, reading notes, Part 1
author: will
date: 2017-05-04T16:27:52+00:00
url: /2017/05/04/designing-data-intensive-application-reading-notes-part-1/
categories:
  - tech
tags:
  - lang-en
  - book
  - Database

---
Notes when reading chapter 2 &#8220;Data models and query languages&#8221;, chapter 3 &#8220;Storage and retrieval&#8221;

<!--more-->

## Data model

### Document model(like mongoDB)

good parts:

  * schema less, do DDL on relational database is painful,
  * can store all related objects in one document(maybe not so good &#8230; refer by id is more flexible)
  * easy to scale in size.

bad parts:

  * hard to describe many-to-many relationship
  * hard to do join
  * lack of transaction

### Graph model

Tow categories:

  1. property graph model, like: Neo4j, Titan
  2. triple-store model, like: Datomic, AllegroGraph. triple means: (subject, predicate, object), eg: (Jim, likes, apple) 

Graph model used in case: everything is potentially related to everything.

Different type of objects are placed in same graph, your query start at a vertex to traverse the graph to get result.

## Storage and retrieval

### SSTable and LSM tree

From Bigtable, opensource: LevelDB, RocksDB.

Insert,update,delete key just append a log on disk, maintain memtable(maybe a B tree) in RAM.

Write operation is fast(sequential writes on disk). Read operation need to check memtable first, if not found, search on disk, from newest sstable to oldest, since data is stored in sorted order, do range query on disk is quick(scan block&#8217;s maxium and minium, like redshift store data by sortkey).

To reclaim disk space, will do compaction for sstable on disk(merge duplicated keys, only keep the latest one).

Tow strategies to decide when to do compaction: size-tiered and leveled.

HBase use size-tired, Cassandra use both, LevelDB and RocksDB use leveled compaction.

size-tired: newer and smaller SSTables are successively merged into older and larger SSTables.

leveled: key range is split up into smaller SSTables and older data is moved into separate &#8220;levels&#8221;, allows the compaction to proceed more incrementally and use less disk space.

downside of LSM:

  * compaction can will cause high write throughput.
  * if write throughput is high, and compaction is not configured carefully, compaction maybe can&#8217;t keep up with incoming write, then the unmerged segments on disk keeps growing.

### B tree

B tree keep key-value pairs sorted by key, allows efficient key-value lookups and range query, this is like LSM tree.

B tree break database into fixed-size blocks or pages, 4 KB(usually), read or write one page at a time. Every page can refer to another, they construct a tree together.

If key number is n, tree depth is O(log n).

LSM is appendonly, but B tree will update pages in place, if the size is over 4 KB, one page will be splitted into two pages, and update their parent page&#8217;s reference. If database crashed during this process, the index will be corrupted. To avoid it, B tree use WAL(write head log) or called redo-log to track every B tree modification, it&#8217;s an append-only file, all operation must be write to it before apply to B tree. If database crashed, it can be used to restore back to a consistent state.

B tree optimization:

  * LMDB don&#8217;t modify page in place, it do it copy-on-write, a modified page is written to a different location and a new version of parent pages in the tree is created.
  * Don&#8217;t store whole key in pages, only provide a few words to act as boundaries between key ranges. Lead to higher branching factor, and fewer level.
  * Try to maintain leaf pages appear in sequential order on disk, then range scan for keys will be efficient.
  * Add pointers for leaf pages, to refer to its sibling pages on left and right, then scaning keys in order needn&#8217;t go back to parent pages. 

### Data warehouse

Stars and snowflakes: fact table and dimension table.

Fact table usually stores events, which is very large in number, some cols are attributes, other columns are foreign keys refer to dimension tables. every row in a fact table is an event.

Dimension tables: user table, product list table&#8230; usually means: who, what, where, how, why &#8230; of the event in fact table.

Data warehouse usually designed as columnar storage, benefits:

  * can only fetch need columns from table (fact table usually have hundreds of columns)
  * can use different compression algorithms for every columns. 

Columnar storage store data on disk by sortkey, # >= 1, advantages:

  * Help narrow down query range quick.
  * If the firt sortkey don&#8217;t have many distinct values, there will be long sequences with same values, then will have great compression ratio.

Columbar storage usually replicate data to different nodes for HA, it&#8217;s possible to use different sort order on replication, when querying, master node can decide choose which version based on query, Vertica did so.
