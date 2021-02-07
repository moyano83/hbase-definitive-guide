# HBase The definitive guide
## Table of contents:
1. [Chapter 1: Introduction](#Chapter1)
2. [Chapter 1: Installation](#Chapter2)



## Chapter 1: Introduction<a name="Chapter1"></a>
### The Dawn of Big Data
HBase stores data on disk in a column-oriented format, it is distinctly different from traditional columnar databases in that whereas columnar databases excel at providing real-time analytical access to data, HBase excels at providing key-based access to a specific cell of data, or a sequential range of cells.

### The Problem with Relational Database Systems
With increasing loads there is several techniques that can be used to cope with the load such as adding slaves, adding a cache for most used queries, scaling servers vertically, denormalize the data for common queries or even prematerialize those queries (compute them in advance), drop secundary indexes...  At the end you have reduced the RDBMS database to just storing your data in a way that is optimized for your access patterns. 

### Nonrelational Database Systems, Not-Only SQL or NoSQL?
A lot of the new kinds of data storage systems do one thing first: throw out the limiting factors in truly scalable systems. For example, they often have no support for transactions or secondary indexes. 

#### Consistency Models
Consistency is about guaranteeing that a database always appears truthful to its clients, thus every operation on the database must carry its state from one consistent state to the next. Consistency can be classified depending on:

    * Strict: The changes to the data are atomic and appear to take effect instantaneously. This is the highest form of consistency.
    * Sequential: Every client sees all changes in the same order they were applied.
    * Causal: All changes that are causally related are observed in the same order by all clients.
    * Eventual: When no updates occur for a period of time, eventually all updates will propagate through the system and all replicas will be consistent.
    * Weak: No guarantee is made that all updates will propagate and changes may appear out of order to various clients.

#### Dimensions
Within the NoSQL category, there are numerous dimensions you could use to classify where the strong points of a particular system lie:

    * Data model: key/value stores, semistructured, column-oriented stores, and document-oriented stores
    * Storage model: In-memory, persistent
    * Consistency model: Strictly or eventually consistent? This may especially affect latency, that is, how fast the system can respond to read and write requests
    * Physical model: Distributed or single machine
    * Read/write performance: Are you designing something that is written to a few times, but is read much more often? Does it support range scans or is it better suited doing random reads? Some of the available systems are advantageous for only one of these operations, while others may do well in all of them
    * Secondary indexes: Secondary indexes allow you to sort and access tables based on different fields and sorting orders.
    * Failure handling: This is related to the “Consistency model”, as losing a machine may cause holes in your data store, or make it completely unavailable
    * Compression: When you have to store terabytes of data, it is advantageous to be able to compress the data to gain substantial savings in required raw storage.
    * Load balancing: Given that you have a high read or write rate, you may want to invest in a storage system that transparently balances itself while the load shifts over time
    * Atomic read-modify-write: Allows you to prevent race conditions in multithreaded or shared-nothing application server design
    * Locking, waits, and deadlocks: It is a known fact that complex transactional processing, like two-phase commits, can increase the possibility of multiple clients waiting for a resource to become available. In a worst-case scenario, this can lead to deadlocks, which are hard to resolve.

#### Scalability
RDBMS are well suited for transactional processing, but not for very large-scale analytical processing (very large queries that scan wide ranges of records or entire tables).

#### Database (De-)Normalization 
The support for sparse, wide tables and column-oriented design often eliminates the need to normalize data and, in the process, the costly JOIN operations needed to aggregate the data at query time. Use of intelligent keys gives you fine-grained control over how—and where—data is stored. 

### Building Blocks
#### Backdrop
HBase is based on BigTable, which is a distributed storage system for managing structured data that is designed to scale to a very large size: petabytes of data across thousands of commodity servers. It is a sparse, distributed, persistent multi-dimensional sorted map.

#### Tables, Rows, Columns, and Cells
The most basic unit is a column. One or more columns form a row that is addressed uniquely by a row key and a number of rows, in turn, form a table. Rows are composed of columns, and those, in turn, are grouped into column families. Each column may have multiple versions, with each distinct value contained in a separate cell. All columns in a column family are stored together in the same low-level storage file, called an HFile. All rows are always sorted lexicographically by their row key (In lexicographical sorting, each key is com- pared on a binary level, byte by byte, from left to right). 
Column families need to be defined when the table is created and the name of the column family must be composed of printable characters. Columns are often referenced as _family:qualifier_ with the qualifier being any arbitrary array of bytes.
Every column value, or cell, either is timestamped implicitly by the system or can be set explicitly by the user. This can be used, to save multiple versions of a value as it changes over time, with different versions of a cell are stored in decreasing time- stamp order, allowing you to read the newest value first. This can be seen as `SortedMap<RowKey, List<SortedMap<Column, List<Value, Timestamp>>>>`.
Access to row data is atomic and includes any number of columns being read or written to. There is no further guarantee or transactional feature that spans multiple rows or across tables. The atomic access is also a contributing factor to this architecture being strictly consistent, as each concurrent reader and writer can make safe assumptions about the state of a row.

#### Auto-Sharding
The basic unit of scalability and load balancing in HBase is called a region. Regions are essentially contiguous ranges of rows stored together. They are dynamically split by the system when they become too large. Alternatively, they may also be merged to reduce their number and required storage files. Initially there is only one region for a table, and as you start adding data to it, the system is monitoring it to ensure that you do not exceed a configured maximum size. If you exceed the limit, the region is split into two at the middle key. Each region is served by exactly one region server, and each of these servers can serve many regions at any time.

#### Storage API
The API offers operations to create and delete tables and column families. In addition, it has functions to change the table and column family metadata, such as compression or block sizes. A scan API allows you to efficiently iterate over ranges of rows and be able to limit which columns are returned or the number of versions of each cell. The system has support for single-row transactions, and with this support it implements atomic read-modify-write sequences on data stored under a single row key. Cell values can be interpreted as counters and updated atomically. There is also the option to run client-supplied code in the address space of the server. The server-side framework to support this is called coprocessors.

#### Implementation
The data is stored in store files, called HFiles, which are persistent and ordered immut- able maps from keys to values. Internally, the files are sequences of blocks with a block index stored at the end. The index is loaded when the HFile is opened and kept in memory (defaults 64 KB). Since every HFile has a block index, lookups can be performed with a single disk seek. When data is updated it is first written to a commit log, called a write-ahead log (WAL) in HBase, and then stored in the in-memory memstore. Once the data in memory has exceeded a given maximum value, it is flushed as an HFile to disk. After the flush, the commit logs can be discarded up to the last unflushed modification. Because store files are immutable, you delete values by adding a delete marker (also known as a tombstone marker) to indicate the fact that the given key has been deleted. Reading data back involves a merge of what is stored in the memstores, that is, the data that has not been written to disk, and the on-disk store files. HBase has a housekeeping mechanism that merges the files into larger ones using compaction. 
There are two types of compaction: minor compactions and major compactions. The former reduce the number of storage files by rewriting smaller files into fewer but larger ones, performing an n-way merge. Since all the data is already sorted in each HFile, that merge is fast and bound only by disk I/O performance. The major compactions rewrite all files within a column family for a region into a single new one. They also have another distinct feature compared to the minor compactions: based on the fact that they scan all key/value pairs, they can drop deleted entries in- cluding their deletion marker. 
There are three major components to HBase: the client library, one master server, and many region servers. The region servers can be added or removed while the system is up and running to accommodate changing workloads. The master is responsible for assigning regions to region servers and uses Apache ZooKeeper.


## Chapter 2: Installation<a name="Chapter2"></a>
