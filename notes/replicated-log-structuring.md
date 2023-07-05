# Replicated log-structuring: seperating indexing from storage for distributed key-value stores

DRAFT IDEA

Sharded key value stores based on replicated logs are the fundamental component of modern distributed
databases (i.e. cockroachdb, tikv, foundationdb, spanner, etc). 

A common design pattern for these stores is to layer a consensus mechanism like Raft (cockroach db, tikv) or Paxos (spanner, foundationdb uses active disk paxos) 
on top of an embedded database engine (cockroachdb uses pebble based on RocksDB, TiKV uses RocksDB, FoundationDB uses sqlite). There is write-amplification
inherent in this strategy, because values must be written both to the replicated log as well as to the database itself. In some cases, additional amplification is
created because of the write-ahead-log (or rollback journal) used for the store. The Spanner paper briefly suggests a future improvement to reduce the write-amplification
of the extra write-ahead-log, but does not discuss reducing the amplificaiton caused by writing to both the log and to the database.

The [WiscKey paper](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf) shows a method for designing embeded kv stores: values can be stored
in a log, while the LSM tree stores only offsets into that log, rather than values. This is faster for very large values (with some tradeoff in compaction/gc overhead).

## work in the embeded (not replicated) database space

Consider that with the increased availability of SSDs there is an opportunity to optimize the design of distributed key-value stores.
There are two relevant properties we considered: while similarily to HDs, SSDs favor sequential writes, SSDs do not
significantly penalize random reads. Secondly, the lifetime of SSD storage is given in terms of write-cycles, and a
key concern with SSDs is reducing write-amplification as well as wear-leveling. Log structuring works well with SSDs:
circular logs evenly wear the disk, and write operations to the log are sequential. Additionally, the strong random read
performance of SSDs means locality matters less to read performance. We can read data scatted across a log-structured
disk relatively cheaply. This is much of the reason why WiscKey works so well.

Another example of an optimized embedded key value database with these principles is the BwTree, which inspired the design 
for the sled database. However, many benefits relate to the case of concurrent transaction handling in a shared-memory
environment. Replicated state machines however do not get many of the concurrency benefits of these systems–as by nature
they must have linearizable behavior.

## what do we need from an embedded database to enable a replicated database?

Let us consider more deeply the disk persistence layer (pebble, a modified rocksdb for cockroachdb;
sqlite for foundation; rocksdb for tikv). A distributed state machine does not require many of the features provided
such as multi-writer concurrency or full transaction support. The experience of CockroachDB suggests that a
persistence layer only primarily support atomic write batches, snapshot isolation, and of course, efficient queries.
The need for a write-ahead-log is also _mostly_ removed, as the replicated log itself can be used for catching up.
Not fully, though. We need to consider how to provide crash consistency unless we assume atomic sector updates. 

## replicated log structuring

We suggest a new seperation of concerns: a Raft object log, and a seperate persistent write-back index (such as an LSM tree
or even more novel indexes). The object log provides inherent point-in-time reads to support MVCC.

This replaces the current practice of seperating the concerns of "replication" from "storage". This proposal comes closer to
"replication and persistent storage of data" and "indexing".

sketch proposal:

Each entry in the Raft log should provide an atomic set of "writes", which we call the "write batch".
Each write is a tuple of `(timestamp, prev_location, identifier, value)`. 

The timestamp likely should be provided by some oracle or hybrid-logical clock, which gives. It should be
the same across all writes in a batch (follows from atomic batches).

The `prev_location` pointer gives the offset in the log of the previous version of the object. This facilitates snapshot
isolation, as a previous version of any object can be found by following pointers backwards.

The identifier provides a way to enable trimming old versions of objects during compaction when desired. See the need
to store keys in the WiscKey value log to enable garbage collection.

The value should be opaque, but there should be a way to mark a tombstone. Similar to the [LLAMA], consider storing deltas
instead of full values.

[LLAMA]: https://www.microsoft.com/en-us/research/publication/llama-a-cachestorage-subsystem-for-modern-hardware/



