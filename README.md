# HBase The definitive guide

## Table of contents:

1. [Chapter 1: Introduction](#Chapter1)
2. [Chapter 1: Installation](#Chapter2)
3. [Chapter 3: Client API: The Basics](#Chapter3)
4. [Chapter 4: Client API: Advanced Features](#Chapter4)

## Chapter 1: Introduction<a name="Chapter1"></a>

### The Dawn of Big Data

HBase stores data on disk in a column-oriented format, it is distinctly different from traditional columnar databases in that
whereas columnar databases excel at providing real-time analytical access to data, HBase excels at providing key-based access
to a specific cell of data, or a sequential range of cells.

### The Problem with Relational Database Systems

With increasing loads there are several techniques that can be used to cope with the load such as adding slaves, adding a
cache for most used queries, scaling servers vertically, denormalize the data for common queries or even prematerialize those
queries (compute them in advance), drop secondary indexes... At the end you have reduced the RDBMS database to just storing
your data in a way that is optimized for your access patterns.

### Nonrelational Database Systems, Not-Only SQL or NoSQL?

A lot of the new kinds of data storage systems do one thing first: throw out the limiting factors in truly scalable systems.
For example, they often have no support for transactions or secondary indexes.

#### Consistency Models

Consistency is about guaranteeing that a database always appears truthful to its clients, thus every operation on the
database must carry its state from one consistent state to the next. Consistency can be classified depending on:

    *Strict: The changes to the data are atomic and appear to take effect instantaneously. This is the highest form of 
      consistency.
    * Sequential: Every client sees all changes in the same order they were applied.
    * Causal: All changes that are causally related are observed in the same order by all clients.
    * Eventual: When no updates occur for a period of time, eventually all updates will propagate through the system and 
      all replicas will be consistent.
    * Weak: No guarantee is made that all updates will propagate and changes may appear out of order to various clients.

#### Dimensions

Within the NoSQL category, there are numerous dimensions you could use to classify where the strong points of a particular
system lie:

    * Data model: key/value stores, semistructured, column-oriented stores, and document-oriented stores
    * Storage model: In-memory, persistent
    * Consistency model: Strictly or eventually consistent? This may especially affect latency, that is, how fast the 
      system can respond to read and write requests
    * Physical model: Distributed or single machine
    * Read/write performance: Are you designing something that is written to a few times, but is read much more often? 
      Does it support range scans or is it better suited doing random reads? Some of the available systems are advantageous 
      for only one of these operations, while others may do well in all of them
    * Secondary indexes: Secondary indexes allow you to sort and access tables based on different fields and sorting orders.
    * Failure handling: This is related to the “Consistency model”, as losing a machine may cause holes in your data 
      store, or make it completely unavailable
    * Compression: When you have to store terabytes of data, it is advantageous to be able to compress the data to gain 
      substantial savings in required raw storage.
    * Load balancing: Given that you have a high read or write rate, you may want to invest in a storage system that 
      transparently balances itself while the load shifts over time
    * Atomic read-modify-write: Allows you to prevent race conditions in multithreaded or shared-nothing application 
      server design
    * Locking, waits, and deadlocks: It is a known fact that complex transactional processing, like two-phase commits, 
      can increase the possibility of multiple clients waiting for a resource to become available. In a worst-case scenario, 
      this can lead to deadlocks, which are hard to resolve.

#### Scalability

RDBMS are well suited for transactional processing, but not for very large-scale analytical processing (very large queries
that scan wide ranges of records or entire tables).

#### Database (De-)Normalization

The support for sparse, wide tables and column-oriented design often eliminates the need to normalize data and, in the
process, the costly JOIN operations needed to aggregate the data at query time. Use of intelligent keys gives you
fine-grained control over how—and where—data is stored.

### Building Blocks

#### Backdrop

HBase is based on BigTable, which is a distributed storage system for managing structured data that is designed to scale to a
very large size: petabytes of data across thousands of commodity servers. It is a sparse, distributed, persistent
multi-dimensional sorted map.

#### Tables, Rows, Columns, and Cells

The most basic unit is a column. One or more columns form a row that is addressed uniquely by a row key and a number of rows,
in turn, form a table. Rows are composed of columns, and those, in turn, are grouped into column families. Each column may
have multiple versions, with each distinct value contained in a separate cell. All columns in a column family are stored
together in the same low-level storage file, called an HFile. All rows are always sorted lexicographically by their row key (
In lexicographical sorting, each key is compared on a binary level, byte by byte, from left to right). Column families need
to be defined when the table is created and the name of the column family must be composed of printable characters. Columns
are often referenced as _family:qualifier_ with the qualifier being any arbitrary array of bytes. Every column value, or
cell, either is timestamped implicitly by the system or can be set explicitly by the user. This can be used, to save multiple
versions of a value as it changes over time, with different versions of a cell are stored in decreasing timestamp order,
allowing you to read the newest value first. This can be seen as
`SortedMap<RowKey, List<SortedMap<Column, List<Value, Timestamp>>>>`. Access to row data is atomic and includes any number of
columns being read or written to. There is no further guarantee or transactional feature that spans multiple rows or across
tables. The atomic access is also a contributing factor to this architecture being strictly consistent, as each concurrent
reader and writer can make safe assumptions about the state of a row.

#### Auto-Sharding

The basic unit of scalability and load balancing in HBase is called a region. Regions are essentially contiguous ranges of
rows stored together. They are dynamically split by the system when they become too large. Alternatively, they may also be
merged to reduce their number and required storage files. Initially there is only one region for a table, and as you start
adding data to it, the system is monitoring it to ensure that you do not exceed a configured maximum size. If you exceed the
limit, the region is split into two at the middle key. Each region is served by exactly one region server, and each of these
servers can serve many regions at any time.

#### Storage API

The API offers operations to create and delete tables and column families. In addition, it has functions to change the table
and column family metadata, such as compression or block sizes. A scan API allows you to efficiently iterate over ranges of
rows and be able to limit which columns are returned or the number of versions of each cell. The system has support for
single-row transactions, and with this support it implements atomic read-modify-write sequences on data stored under a single
row key. Cell values can be interpreted as counters and updated atomically. There is also the option to run client-supplied
code in the address space of the server. The server-side framework to support this is called coprocessors.

#### Implementation

The data is stored in store files, called HFiles, which are persistent and ordered immutable maps from keys to values.
Internally, the files are sequences of blocks with a block index stored at the end. The index is loaded when the HFile is
opened and kept in memory (defaults 64 KB). Since every HFile has a block index, lookups can be performed with a single disk
seek. When data is updated it is first written to a commit log, called a write-ahead log (WAL) in HBase, and then stored in
the in-memory memstore. Once the data in memory has exceeded a given maximum value, it is flushed as an HFile to disk. After
the flush, the commit logs can be discarded up to the last unflushed modification. Because store files are immutable, you
delete values by adding a delete marker (also known as a tombstone marker) to indicate the fact that the given key has been
deleted. Reading data back involves a merge of what is stored in the memstores, that is, the data that has not been written
to disk, and the on-disk store files. HBase has a housekeeping mechanism that merges the files into larger ones using
compaction. There are two types of compaction: minor compactions and major compactions. The former reduce the number of
storage files by rewriting smaller files into fewer but larger ones, performing an n-way merge. Since all the data is already
sorted in each HFile, that merge is fast and bound only by disk I/O performance. The major compactions rewrite all files
within a column family for a region into a single new one. They also have another distinct feature compared to the minor
compactions: based on the fact that they scan all key/value pairs, they can drop deleted entries including their deletion
marker. There are three major components to HBase: the client library, one master server, and many region servers. The region
servers can be added or removed while the system is up and running to accommodate changing workloads. The master is
responsible for assigning regions to region servers and uses Apache ZooKeeper.

## Chapter 2: Installation<a name="Chapter2"></a>

### Quick-Start Guide

You can download a tar file containing the latest hbase code and run it prior setting the data path specified in the file
_hbase-site.xml_ and setting the property _hbase.tmp.dir_, once it is changed you can start the service
`./bin/start-hbase.sh`, the _JAVA\_HOME_  variable must be set. In mac, use `export JAVA_HOME=$(/usr/libexec/java_home)`
to set this variable. To start an hbase shell, use `./bin/hbase shell` and then you can check the status. To create a test
table, execute `create 'testtable', 'colfam1'` and use `list` to view the newly created table. Use the following command to
add data:

```hbase
hbase:003:0> put 'testtable', 'myrow-1', 'colfam1:q1', 'value-1'
hbase:004:0> put 'testtable', 'myrow-2', 'colfam1:q2', 'value-2'
hbase:005:0> put 'testtable', 'myrow-2', 'colfam1:q3', 'value-3'
```

The result of the above is two rows, _myrow-1_ and _myrow-2_, the first with a column _q1_ the second with two _q2_ and _q3_.
You can view the results with `scan 'testtable'` or if we just want one cell back, we can use `get 'testtable', 'myrow-1'`.
Delete a value using delete `'testtable', 'myrow-2', 'colfam1:q2'` and a table using disable plus drop table and then exit
the shell like this:

```hbase
hbase(main):011:0> disable 'testtable'
hbase(main):012:0> drop 'testtable'
hbase(main):012:0> exit
```

### Requirements

#### Hardware

HBase is written in Java, so at least you need support for a current Java Runtime and a 64-bit operating system to be able to
address enough memory. In HBase and Hadoop there are two types of machines: masters and slaves. The master does not need that
much disk but it needs to be a powerfull and have some redundant components is beneficial for failover. HBase use-cases are
mostly I/O bound, so having more cores will help keep the data drives busy.

#### Software

You must set _JAVA\_HOME_ on each node of your cluster, the _conf/hbase-env.sh_ script provides a mechanism to do this. HBase
depends on Hadoop, it bundles an instance of the Hadoop JAR under its lib directory. It is important that the version of
Hadoop that is in use on your cluster matches what is used by HBase.

#### Filesystems for HBase

The FileSystem used by HBase has a pluggable architecture and can be used to replace HDFS with any other supported system.
HBase has no added means to replicate data or even maintain copies of its own storage files. This functionality must be
provided by the lower-level system. There are several choices for filesystems:

    * Local: Used by the standalone mode, select it using the scheme `file:///<path>`
    * HDFS: Default when using with a cluster, select it using the scheme `hdfs://<namenode>:<port>/<path>`
    * S3: The S3 FileSystem implementation provided by Hadoop supports three different modes: the raw mode, which uses 
      the `s3n` URI scheme, the block-based mode which uses the `s3` scheme and the AWS SDK based mode which uses the `s3a:`
      scheme

### Installation Choices

The contents of the binary file that you can download from apache repositories are the following:

    * bin: Binaries folder, contains the scripts to run/stop the application among others
    * conf: Configuration files
    * docs: Documentation of the project
    * hbase-webapps: HBase has web-based user interfaces which are implemented as Java web applications
    * lib: Java libraries required by hbase
    * logs: Logs of the running application, initially empty

### Run modes

HBase has two run modes: standalone and distributed. To set up HBase in distributed mode, you will need to edit files in the
HBase conf directory.

#### Standalone mode

This is the default mode, it does not use HDFS but the local filesystem instead, and it runs all HBase daemons and a local
ZooKeeper in the same JVM process.

#### Distributed mode

The distributed mode can be further subdivided into pseudodistributed in where all daemons run on a single node, and fully
distributed, where the daemons are spread across multiple, physical servers in the cluster. It uses HDFS.

##### Pseudo-distributed mode

Use this configuration for testing and prototyping on HBase. Do not use this configuration for production or for evaluating
HBase performance. To use this configuration, edit _conf/hbase-site.xml_ and point HBase at the running Hadoop HDFS instance
by setting the _hbase.rootdir_ property. i.e.

```xml

<configuration>
    <!-- Some other config -->
    <property>
        <name>hbase.rootdir</name>
        <value>hdfs://localhost:9000/hbase</value>
    </property>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <!-- Some other config -->
</configuration>
```

##### Fully distributed mode

To use this mode, edit _hbase-site.xml_ and add the _hbase.cluster.distributed_ property and set it to 'true' and then point
the HBase _hbase.rootdir_ at the appropriate HDFS name node and location in HDFS where you would like HBase to write data.
i.e.

```xml

<configuration>
    <!-- Some other config -->
    <property>
        <name>hbase.rootdir</name>
        <value>hdfs://namenode.foo.com:9000/hbase</value>
    </property>
    <property>
        <name>hbase.cluster.distributed</name>
        <value>true</value>
    </property>
    <!-- Some other config -->
</configuration>
```

A fully distributed mode requires that you modify the _conf/regionservers_ file. It lists all the hosts on which you want to
run HRegionServer daemons (one host per line, initially contains just a line with 'localhost'). A distributed HBase setup
also depends on a running ZooKeeper cluster.

All participating nodes and clients need to be able to access the running ZooKeeper ensemble. HBase, by default, manages a
ZooKeeper cluster and it starts and stops the ZooKeeper ensemble as part of the HBase start and stop process. You can also
manage the ZooKeeper ensemble independent of HBase and just point HBase at the cluster it should use. To toggle HBase
management of ZooKeeper, use the _HBASE\_MANAGES\_ZK_ variable in conf/hbase-env.sh. This variable, which defaults to true,
tells HBase whether to start and stop the ZooKeeper ensemble servers as part of the start and stop commands supplied by
HBase. When HBase manages the ZooKeeper ensemble, you can specify the ZooKeeper configuration options directly in
_conf/hbase-site.xml_. You can set a ZooKeeper configuration option as a property in the _hbase-site.xml_ configuration file
by prefixing the ZooKeeper option name with _hbase.zookeeper.property_. You must at least set the ensemble servers with the
_hbase.zookeeper.quorum_ property, otherwise it defaults to single node ensemble. Some important properties are:

    * zookeeper.: Specifies client settings for the ZooKeeper client used by the HBase client library
    * hbase.zookeeper.: Used for values pertaining to the HBase client communicating to the ZooKeeper servers
    * hbase.zookeeper.properties: These are only used when HBase is also managing the ZooKeeper ensemble

To point HBase at an existing ZooKeeper cluster, set _HBASE\_MANAGES\_ZK_ in _conf/hbase-env.sh_ to false. Next, set the
ensemble locations and client port (if nonstandard), in _hbase-site.xml_.

### Configuration

The following is a list of the most important configuration files for hbase:

    * hbase-env.sh: Set up the working environment for HBase
    * hbase-site.xml: The main configuration file. It specifies configuration options which override HBase’s defaults
    * backup-masters: This is a text file that lists all the hosts which should have backup masters started on
    * regionservers: Lists all the nodes that are designated to run a region server instance
    * hadoop-metrics2-hbase.properties: Specifies settings for the metrics framework integrated into each HBase process
    * hbase-policy.xml: In secure mode, this file defines the authorization rules for clients accessing the servers
    * log4j.properties: Configures how each process logs its information using the Log4J libraries

When running in distributed mode, after you make an edit to a HBase configuration file, make sure you copy the content of the
conf directory to all nodes of the cluster. HBase will not do this for you. The servers always read the hbase-default.xml
file first and subsequently merge it with the hbase-site.xml file content (if present). Most changes here will require a
cluster restart for HBase to notice the change. However, there is a way to reload some specific settings while the processes
are running. Properties in this two files takes precedence over any Hadoop configuration file containing a property with the
same name.

#### Client Configuration

Since the HBase Master may move around between physical machines, clients start by requesting the vital information from
ZooKeeper, therefore clients require the ZooKeeper quorum information in a _hbase-site.xml_ file on their Java _$CLASSPATH_.

### Deployment

Several approaches can be taken to perform deployment of the hbase distribution into a cluster, for example:

    * Script-based deployment: Appropriate to a small/medium size cluster only
    * Apache Whirr: Suitable for running HBase in dynamic environments like AWS or google cloud. Whirr has support for a 
      variety of public and private cloud APIs and allows you to provision clusters running a range of services
    * Puppet and Chef: Similar to Whirr, this two frameworks has a central provisioning server that stores all the 
      configurations, combined with client software, executed on each server, which communicates with the central 
      server to receive updates and apply them locally

### Operating a Cluster

### Web-based UI Introduction

HBase starts a web-based UI listing vital attributes, which is deployed on the master host at port 16010 (default). This page
has information about available region servers, as well as any optional backup masters. This is followed by the known tables,
system tables, and snapshots. It also contains information about currently running tasks, for example, the RPC handler
status, active calls, and so on. Finally the bottom of the page has the attributes pertaining to the cluster setup.

#### Shell Introduction

The HBase Shell is (J)Ruby’s IRB with some HBase-related commands added. Find more by typing _help_ to see a listing of shell
commands and options.

## Chapter 3: Client API: The Basics<a name="Chapter3"></a>

### General notes

The primary client entry point to HBase is the _Table_ interface in the _org.apache.hadoop.hbase.client_ package which is
retrieved by means of the _Connection_ instance. All operations that mutate data are guaranteed to be atomic on a per-row
basis (irrespective of the number of columns. As a general rule, try to batch updates together to reduce the number of
separate operations on the same row as much as possible. Creating an initial connection to HBase is heavy as each
instantiation involves scanning the _hbase:meta_ table to check if the table actually exists and if it is enabled, as well as
a few other operations.

Create a _Connection_ instance only once and reuse it for the rest of the lifetime of your client application. Once you have
a connection instance you can retrieve references to the actual tables. Ideally you do this per thread since the underlying
implementation of Table is not guaranteed to the thread-safe. Make sure to close the connection once you are done.

### Data Types and Hierarchy

There are several data centric classes (operations):

    * Get (Query): Retrieve previously stored data from a single row
    * Scan (Query): Iterate over all or specific rows and return their data
    * Put (Mutation): Create or update one or more columns in a single row
    * Delete (Mutation): Remove a specific cell, column, row, etc.
    * Increment (Mutation): Treat a column as a counter and increment its value
    * Append (Mutation): Attach the given data to one or more columns in a single row

#### Generic Attributes

The interface _Attribute_ provides a general mechanism to add any kind of information in form of attributes to all the
data-centric classes. It has the following methods:

```
Attributes setAttribute(String name,byte[]value);
byte[]getAttribute(String name);
Map<String, byte[]>getAttributesMap();
```

#### Operations: Fingerprint and ID

Another fundamental type is the abstract class Operation, which adds the following methods to all data types:

```
// Returns the list of column families included in the instance
abstract Map<String, Object> getFingerprint() 

// Compiles a list including fingerprint, column families with all columns and their data, 
// total column count, row key, and—if set—the ID and cell-level TTL
abstract Map<String, Object> toMap(int maxCols) 

// Same as above, but only for 5 columns
Map<String, Object> toMap()

// Same as toMap(maxCols) but converted to JSON. Might fail due to encoding issues
String toJSON(int maxCols) throws IOException

// Same as above, but only for 5 columns
String toJSON() throws IOException

// Attempts to call toJSON(maxCols), but when it fails, falls back to toMap(maxCols)
String toString(int maxCols)

// Same as above, but only for 5 columns
String toString()
```

This class helps in generating useful information collections for logging and general debugging purposes. The intermediate
OperationWithAttributes class is extending the above Operation class, implements the Attributes interface, and is adding the
following methods, which are used in conjunction:

```
OperationWithAttributes setId(String id)

// Returns what was set by the setId() method
String getId() 
```

#### Query versus Mutation

_Row_ is also an important interface with the following methods:

```
// returns the given row key of the instance, implemented by the Query and Mutation classes
byte[] getRow()
```

_Mutation_ implements the _CellScannable_ interface to provide the following method:

```
// A client can call this method to iterate over the returned cells
CellScanner cellScanner()
```

Other methods in the mutation class:

    * getACL()/setACL(): The Access Control List (ACL) for this operation
    * getCellVisibility()/setCellVisibility(): The cell level visibility for all included cells
    * getClusterIds()/setClusterIds(): The cluster ID as needed for replication purposes
    * getDurability()/setDurability(): The durability settings for the mutation
    * getFamilyCellMap()/setFamilyCellMap(): The list of all cells per column family available in this instance
    * getTimeStamp(): Retrieves the associated timestamp of the Put instance
    * getTTL()/setTTL(): Sets the cell level TTL value. It is applied to all included Cell instances before being persisted
    * isEmpty(): Checks if the family map contains any Cell instances
    * numFamilies(); Convenience method to retrieve the size of the family map, containing all Cell instances
    * size(): Returns the number of Cell instances that will be added with this Put
    * heapSize(): Computes the heap space required for the current Put instance. This includes all contained data and space
      needed for internal structures

The other larger superclass on the retrieval side is Query, which provides a common substrate for all data types concerned
with reading data from the HBase tables:

    * getAuthorizations()/setAuthorizations(): Visibility labels for the operation
    * getACL()/setACL(): The Access Control List (ACL) for this operation
    * getFilter()/setFilter(): The filters that apply to the retrieval operation
    * getConsistency()/setConsistency(): The consistency level that applies to the current query instance
    * getIsolationLevel()/setIsolation Level(): Specifies the read isolation level for the operation
    * getReplicaId()/setReplicaId(): Gives access to the replica ID that served the data

#### Durability, Consistency, and Isolation

Some properties set on read/write operation makes use of enums to set some specifics about the operation itself. For example
to set the durability levels:

    * USE_DEFAULT: For tables use the global default setting, which is SYNC_WAL. For a mutation use the table’s default value 
    * SKIP_WAL: Do not write the mutation to the WAL
    * ASYNC_WAL: Write the mutation asynchronously to the WAL 
    * SYNC_WAL: Write the mutation synchronously to the WAL 
    * FSYNC_WAL: Write the Mutation to the WAL synchronously and force the entries to disk

Durability lets you decide how important your data is to you, just because the client library you are using accepts the
operation does not imply that it has been applied, or persisted even.

On the read side, we can control how consistent is the data read, the options are:

    * STRONG: Strong consistency as per the default of HBase. Data is always current
    * TIMELINE: Replicas may not be consistent with each other, updates are guaranteed to be applied in order in all replicas

HBase always writes and commits all changes strictly serially, which means that completed transactions are always presented
in the exact same order. The `isStale()` method is used to check if we have retrieved data from a replica, not the
authoritative master.

On the read side, you can also set the isolation level (on the Query superclass). The options are:

    * READ_COMMITTED: Read only data that has been committed by the authoritative server
    * READ_UNCOMMITTED: Allow reads of data that is in flight, i.e. not committed yet

#### The Cell

A cell instance contains the data as well as the coordinates (row key, name of the column family, column qualifier, and
timestamp) of one specific cell. _Cell_ is an interface, but the implementing class, named _KeyValue_ is private and cannot
be instantiated either. The _CellUtil_ class, among many other convenience functions, provides the necessary methods to
create an instance for us. The data as well as the coordinates are stored as a `Java byte[]` to allow for any arbitrary data
and to efficiently store only the required bytes, keeping the overhead of internal data structures to a minimum (hence there
is an Offset and Length parameter for each byte array parameter). A _Cell_ has a type, which can be one of `Put`,`Delete`,
`DeleteFamilyVersion` (deletes all columns of a column family matching a specific timestamp), `DeleteColumn`,
`DeleteFamily`. The `toString()` method of a cell instance, prints the following information:
`<row-key>/<family>:<qualifier>/<version>/<type>/<value-length>/<sequence-id>`.

The _CellComparator_ class is the base to classes that compare given cell instances using the Java _Comparator_ pattern. One
class is publicly available as an inner class of _CellComparator_, namely the _RowComparator_. You can use this class to
compare cells by the given row key.

#### API Building Blocks

The basic flow of a client connecting and calling the API looks like this:

```
// Hadoop class, shared by HBase, to load and provide the configuration to the client application. This class will attempt to 
// load two configuration files, hbase-default.xml and hbase-site.xml, using the current Java class path
Configuration conf=HBaseConfiguration.create();
// Factory method to retrieve a Connection instance,
Connection connection=ConnectionFactory.createConnection(conf);
// TableName represents a table name with its namespace
TableName tableName=TableName.valueOf("testtable");
// The lightweight, not thread-safe representation of a data table within the client API
Table table=connection.getTable(tableName);

Result result=table.get(get);

// You need to close the table and connection to free resources
table.close();
connection.close();
```

##### Resource Sharing

Every instance of _Table_ requires a connection to the remote servers, which is handled by the _Connection_ implementation
instance, acquired using the _ConnectionFactory_. Reuse the connection as every connection does a lot of internal resource
handling like:

    * Share ZooKeeper Connections: Initial lookup of where user table regions are located
    * Cache Common Resources: Every zookeeper lookup requires a round network call, but the location is then cached on 
      the client side to reduce the amount of network traffic, and to speed up the lookup process, if this lookup fails, 
      the connection has a built-in retry mechanism which result is passed to all other application threads sharing the 
      same connection reference

### CRUD Operations

CRUD operations are provided by the _Table_ interface.

#### Put Method

Put operations can be split into those that work on single rows, those that work on lists of rows, and one that provides a
server-side, atomic check-and-put.

##### Single Puts

This is the general put operation: `void put(Put put) throws IOException` which depending on the constructor used, would have
a different effect. You need to supply a row to create a _Put_ instance, and a row in HBase is identified by a unique row
key (a Java byte[] array). HBase provide us with a helper class that has many static methods to convert Java types into
byte[] arrays. i.e. `byte[] rowkey = Bytes.toBytes("row_key_id");`. there are also constructor variants that take an existing
byte array and, respecting a given offset and length parameter, copy the row key bits from the given array instead. i.e:

```
System.arraycopy("user".getBytes(Charset.forName("UTF8"), 0, dest_array, 45, user‐name_bytes.length);
Put put = new Put(data, 45, username_bytes.length);
```

There are as well constructors with buffers and even other _Put_ instances (to clone it). Once you have the _Put_
instance, you can add data to it. Each call to `addColumn()` specifies exactly one column, or, in combination with an
optional timestamp, one single cell. Calling any of the `addXX()` methods in a _Put_ instance will internally create a _Cell_
instance. There are copies of each `addColumn()`, named `addImmutable()`, which do the same as their counterpart, apart from
not copying the given byte arrays. The variant that takes an existing _Cell_ instance is for users that knows how to
retrieve, or create, this low-level class. To check for the existence of specific cells, you can use any variant of
the `boolean has(...)` method, which returns true if a match is found.

    * cellScanner(): Provides a scanner over all cells available in this instance
    * getACL()/setACL(): The ACLs for this operation 
    * getAttribute()/setAttribute(): Set and get arbitrary attributes associated with this instance of Put
    * getAttributesMap(): Returns the entire map of attributes, if any are set
    * getCellVisibility()/setCellVisibility(): The cell level visibility for all included cells
    * getClusterIds()/setClusterIds(): The cluster IDs as needed for replication purposes
    * getDurability()/setDurability(): The durability settings for the mutation
    * getFamilyCellMap()/setFamilyCellMap(): The list of all cells of this instance
    * getFingerprint(): Compiles details about the instance into a map for debugging, or logging
    * getId()/setId(): An ID for the operation, useful for identifying the origin of a request later
    * getRow(): Returns the row key as specified when creating the Put instance
    * getTimeStamp(): Retrieves the associated timestamp of the Put instance
    * getTTL()/setTTL(): Sets the cell level TTL value, which is being applied to all included Cells before being persisted
    * heapSize(): Computes the heap space required for the current Put instance. Including data and internal structures
    * isEmpty(): Checks if the family map contains any Cell instances
    * numFamilies(): Convenience method to retrieve the size of the family map, containing all Cell instances
    * size(): Returns the number of Cell instances that will be applied with this Put 
    * toJSON()/toJSON(int): Converts the first 5 or N columns into a JSON format
    * toMap()/toMap(int): Converts the first 5 or N columns into a map. More detailed than what getFingerprint() returns 
    * toString()/toString(int): Converts the first 5 or N columns into JSON, or map (if JSON fails due to encoding problems)

Example:

```
Connection connection = ConnectionFactory.createConnection(HBaseConfiguration.create()); 
Table table = connection.getTable(TableName.valueOf("testtable"));
Put put = new Put(Bytes.toBytes("row1"));
put.addColumn(Bytes.toBytes("colfam1"), Bytes.toBytes("qual1"), Bytes.toBytes("val1"));
table.put(put);
table.close();
connection.close();
```

##### Client-side Write Buffer

Each _Put_ operation is a remote producure call (RPC) which transfers data from the client to the server and back. To reduce
the number of round trips and hence response time, the HBase API comes with a built-in client-side write buffer that collects
put and delete operations so that they are sent in one RPC call to the server(s). The entry point to this functionality is
the _BufferedMutator_ class (thread safe). It is obtained from the _Connection_ class using one of these methods:

```
BufferedMutator getBufferedMutator(TableName tableName) throws IOException
BufferedMutator getBufferedMutator(BufferedMutatorParams params) throws IOException
```

Important considerations about this class includes:

    * Call close() at the very end of its lifecycle
    * It might be necessary to call flush() when you have submitted mutations that need to go to the server immediately
    * If flush is not called, then the update happens asynchronously when the threshold is reached, or when close() is called
    * Local mutations that are still cached could be lost if the application fails

A _BufferedMutator_ instance can be customized through a _BufferedMutatorParams_ class, which contains useful methods like:

```
BufferedMutatorParams(TableName tableName) // Constructor
TableName getTableName() // Returns the table name
long getWriteBufferSize() // In case the WriteBufferSize is exceeded, the cached mutations are sent asynchronously
BufferedMutatorParams writeBufferSize(long writeBufferSize)
int getMaxKeyValueSize() // the size of the included cells is checked against the upper limit set in MaxKeyValueSize setting
BufferedMutatorParams maxKeyValueSize(int maxKeyValueSize)
ExecutorService getPool() // The pool to execute the operations asynchronously
BufferedMutatorParams pool(ExecutorService pool)
BufferedMutator.ExceptionListener getListener() // listener to be notified when on errors during the mutation on the servers
BufferedMutatorParams listener(BufferedMutator.ExceptionListenerlistener)
```

An example of buffer mutator is below:

```
TableName name = Table table = connection.getTable(TableName.valueOf("testtable"));
Connection connection = ConnectionFactory.createConnection(conf);
BufferedMutator mutator = connection.getBufferedMutator(name);
Put put1 = new Put(Bytes.toBytes("row1"));
put1.addColumn(Bytes.toBytes("colfam1"), Bytes.toBytes("qual1"), Bytes.toBytes("val1"));
mutator.mutate(put1);
...

mutator.flush();

mutator.close();
table.close();
connection.close();    
```

##### List of Puts

It is also possible to batch a set of put operations through the method `void put(List<Put> puts) throws IOException` of
the _Table_ class:

```
List<Put> puts = new ArrayList<Put>();
...
table.put(puts);
```

Since you are issuing a list of row mutations to possibly many rows, there is a chance that not all of them will succeed. If
this happens, an _IOException_ is thrown. The servers iterate over all operations and try to apply them. The failed ones are
returned, and the client reports the remote error using the _RetriesExhaustedWithDetailsException_, giving you insight into
how many operations have failed, with what error, and how many times it has retried to apply the erroneous modification. Some
methods of the _RetriesExhaustedWithDetailsException_ class:

    * getCauses(): Returns a summary of all causes for all failed operations
    * getExhaustiveDescription(): More detailed list of all the failures that were detected
    * getNumExceptions(): Returns the number of failed operations
    * getCause(int i): Returns the exact cause for a given failed operation
    * getHostnamePort(int i): Returns the exact host that reported the specific error
    * getRow(int i): Returns the specific mutation instance that failed
    * mayHaveClusterIssues(): Allows to determine if there are wider problems with the cluster

The MaxKeyValueSize check is done as well for a list of puts. The list-based `put()` call uses the client-side write buffer
in form of an internal instance of _BatchMutator_ to insert all puts into the local buffer and then to call `flush()`
implicitly. While inserting each instance of Put, the client API performs the mentioned check. If it fails, for example, at
the third put out of five, the first two are added to the buffer while the last two are not, and the flush command is not
triggered. You need to keep inserting puts in the list or call the close() to trigger a flush. Also, you cannot control the
order in which the puts are applied on the server-side. Because of this, _BufferedMutator_ is preferred over the put list on
the _Table_ class.

##### Atomic Check-and-Put

The method signatures are:

```
boolean checkAndPut(byte[] row, byte[] family, byte[] qualifier, byte[] value, Put put) throws IOException
boolean checkAndPut(byte[] row, byte[] family, byte[] qualifier, CompareFilter.CompareOp compareOp, byte[] value, Put put) throws IOException
```

These calls allow you to issue atomic, server-side mutations that are guarded by an accompanying check. If the check passes
successfully, the put operation is executed; otherwise, it aborts the operation completely. The first call implies that the
given value has to be equal to the stored one. The second call lets you specify the actual comparison operator. An usage
example of this is to update if another value is not already present by setting the _value_ parameter to _null_. The call
returns a boolean result value, indicating whether the Put has been applied or not. atomicity guarantees applies only on
single rows.

#### Get Method

This operations is split into single or multiple row retrieval.

##### Single Gets

the method that is used to retrieve specific values from a HBase table: `Result get(Get get) throws IOException`. A `get()`
operation is bound to one specific row, but can retrieve any number of columns and/or cells contained therein.  
Constructors of _Get_ class:

```
Get(byte[] row) // row is the row  key to retrieve
Get(Get get) // Clones the content of the get
```

To narrow the search or to specify everything down to exact coordinates for a single cell, you have the following methods:

```
Get addFamily(byte[] family) // To narrow the search to the column family
Get addColumn(byte[] family, byte[] qualifier) // To narrow the search to the column family and column
Get setTimeRange(long minStamp, long maxStamp) throws IOException // sets a time range for the search
Get setTimeStamp(long timestamp) // Sets a specific timestamp
Get setMaxVersions() // sets the number of versions to return to Integer.MAX_VALUE
Get setMaxVersions(int maxVersions) throws IOException
```

Example of the usage is displayed below:

```
// Create connection
Configuration conf = HBaseConfiguration.create();
Connection connection = ConnectionFactory.createConnection(conf); 
Table table = connection.getTable(TableName.valueOf("testta‐ble"));
// Instantiate Get
Get get1 = new Get(Bytes.toBytes("row1"));
get.addColumn(Bytes.toBytes("colfam1"), Bytes.toBytes("qual1"));

// There is a fluent Api for this as well
Get get2 = new Get(Bytes.toBytes("row1"))
      .setId("GetFluentExample")
      .setMaxVersions()
      .setTimeStamp(1)
      .addColumn(Bytes.toBytes("colfam1"), Bytes.toBytes("qual1"))
      .addFamily(Bytes.toBytes("colfam2"));

// Retrieve results
Result result = table.get(get1);
byte[] val = result.getValue(Bytes.toBytes("colfam1"), Bytes.toBytes("qual1"));
// Close connection
table.close();
connection.close();
```

Additional methods of the _Get_ class:

    * familySet()/getFamilyMap(): Give you access to the column families and specific columns,
    * getACL()/setACL(): The Access Control List (ACL) for this operation.
    * getAttribute()/setAttribute(): Set and get arbitrary attributes associated with this instance of Get
    * getAttributesMap(): Returns the entire map of attributes, if any are set
    * getAuthorizations()/setAuthorizations(): Visibility labels for the operation 
    * getCacheBlocks()/setCacheBlocks(): Specify if the server-side cache should retain blocks that were loaded for this 
      operation
    * setCheckExistenceOnly()/is CheckExistenceOnly(): Only check for existence of data, but do not return any of it
    * setClosestRowBefore()/isClosestRowBefore(): Return all the data for the row that matches the given row key exactly, or 
      the one that immediately precedes it 
    * getConsistency()/setConsistency(): The consistency level that applies to the current query instance
    * getFilter()/setFilter(): The filters that apply to the retrieval operation
    * getFingerprint(): Compiles details about the instance into a map for debugging, or logging 
    * getId()/setId(): An ID for the operation, useful for identifying the origin of a request later
    * getIsolationLevel()/setIsolationLevel(): Specifies the read isolation level for the operation
    * getMaxResultsPerColumnFamily()/setMaxResultsPerColumnFamily(): Limit the number of cells returned per family
    * getMaxVersions()/setMaxVersions(): Override the column family setting specifying how many versions of a column to get
    * getReplicaId()/setReplicaId(): Gives access to the replica ID that should serve the data
    * getRow(): Returns the row key as specified when creating the Get instance
    * getRowOffsetPerColumnFamily()/setRowOffsetPerColumnFamily(): Number of cells to skip when reading a row
    * getTimeRange()/setTimeRange(): Retrieve or set the associated timestamp or time range of the Get instance 
    * setTimeStamp(): Sets a specific timestamp for the query. Retrieve with getTimeRange()
    * numFamilies(): Retrieves the size of the family map, containing families added using the addFamily()/addColumn() calls
    * hasFamilies(): Another helper to check if a family—or column—has been added to the current instance of the Get class
    * toJSON()/toJSON(int): Converts the first 5 or N columns into a JSON format
    * toMap()/toMap(int): Converts the first 5 or N columns into a map
    * toString()/toString(int): Converts the first 5 or N columns to a JSON, or map (if JSON fails due to encoding problems)

`setCacheBlocks()` and `getCacheBlocks()` controls how the read operation is handled on the server-side. Each HBase region
server has a block cache that efficiently retains recently accessed data for subsequent reads of contiguous information. In
some events it is better to not engage the cache to avoid too much churn when doing completely random gets.
`setCheckExistenceOnly()` and `isCheckExistenceOnly()` combination allows the client to check if a specific set of columns,
or column families are already existent. If multiple checks exists, they are added with the or operator (returns true if one
exists). The _Table_ class has another way of checking for the existence of data in a table:

```
boolean exists(Get get) throws IOException
boolean[] existsAll(List<Get> gets) throws IOException;
```

`setClosestRowBefore()` and `isClosestRowBefore()` offers some sort of fuzzy matching for rows. Using `setClosestRowBefore()`
would return the previous row to the one you are looking specifically if it does not find a match. In the case this  
behaviour is triggered, the entire row is returned and not only a set of columns.
`getMaxResultsPerColumnFamily()`, `setMaxResultsPerColumnFamily()`, `getRowOffsetPerColumnFamily()`, and
`setRowOffsetPerColumnFamily()` works in tandem to allow the client to page through a wide row, setting the maximum amount of
cells returned by a get request or an optional offset into the row:

```
Get get1 = new Get(Bytes.toBytes("row1"));
get1.setMaxResultsPerColumnFamily(10);
Result result1 = table.get(get1);
CellScanner scanner1 = result1.cellScanner();
while (scanner1.advance()) {
    System.out.println("Get 1 Cell: " + scanner1.current());
}
```

The above really works in cells, not columns, so remember that when iterating over the results.

##### The Result class

The result class provides you with the means to access everything that was returned from the server for the given row and
matching the specified query, such as column family, column qualifier, timestamp, and so on. Some of the methods:

```
byte[] getRow()
byte[] getValue(byte[] family, byte[] qualifier) // returns the newest value of a cell
byte[] value() // returns the data for the newest cell in the first column found (columns are also sorted lexicographically)
ByteBuffer getValueAsByteBuffer(byte[] family, byte[] qualifier) 
ByteBuffer getValueAsByteBuffer(byte[] family, int foffset, int flength,byte[] qualifier, int qoffset, int qlength)
boolean loadValue(byte[] family, byte[] qualifier, ByteBuffer dst) throws BufferOverflowException
boolean loadValue(byte[] family, int foffset, int flength, byte[] qualifier, int qoffset, int qlength, ByteBuffer dst) 
        throws BufferOverflowException
CellScanner cellScanner()
Cell[] rawCells() // returns the array of Cell instances backing the current Result instance
List<Cell> listCells() // same than before but in a List format
boolean isEmpty()
int size()
```

Some of the methods to return data clone the underlying byte array so that no modification is possible. Yet others do not and
you have to take care not to modify the returned arrays. Some of the methods in the Result class are more column oriented,
for example `getColumnCells()` method you ask for multiple values of a specific column, allowing you to get multiple versions
of a given column. Methods that operates in columns are described below:

```
List<Cell> getColumnCells(byte[] family, byte[] qualifier)
Cell getColumnLatestCell(byte[] family, byte[] qualifier) // returs the newest cell of the specified column
Cell getColumnLatestCell(byte[] family, int foffset, int flength, byte[] qualifier, int qoffset, int qlength)
boolean containsColumn(byte[] family, byte[] qualifier)
boolean containsColumn(byte[] family, int foffset, int flength, byte[] qualifier, int qoffset, int qlength)
boolean containsEmptyColumn(byte[] family, byte[] qualifier) 
boolean containsEmptyColumn(byte[] family, int foffset, int flength,byte[] qualifier, int qoffset, int qlength)
boolean containsNonEmptyColumn(byte[] family, byte[] qualifier) 
boolean containsNonEmptyColumn(byte[] family, int foffset, int flength, byte[] qualifier, int qoffset, int qlength)
```

There is a third set of methods that provide access to the returned data, and are map-oriented:

```
// returns the entire result set in a Java Map class, here you get only the data in a map, not any accessors or other 
// internal information of the cells
NavigableMap<byte[], NavigableMap<byte[], NavigableMap<Long, byte[]>>> getMap()  
// only includes the latest cell for each column 
NavigableMap<byte[],  NavigableMap<byte[],  byte[]>>  getNoVersionMap()
// lets you select the data for a specific column family only
NavigableMap<byte[], byte[]> getFamilyMap(byte[] family)
```

Other methods of the Result class:

```
create() // There is a set of these static methods to help create Result instances if necessary
copyFrom() // Helper method to copy a reference of the list of cells from one instance to another
compareResults() // Static method, does a deep compare of two instance, down to the byte arrays
getExists()/setEx ists() // Optionally used to check for existence of cells only
getTotalSizeOfCells() // Static method, summarizes the estimated heap size of all contained cells. Uses Cell.heapSize() for each contained cell.
isStale() // Indicates if the result was served by a region replica, not the main one
addResults()/get Stats() // This is used to return region statistics, if enabled (default is false).
toString() // Dump the content of an instance for logging or debugging
```

##### List of Gets

Another similarity to the `put()` calls is that you can ask for more than one row using a single request with:
`Result[] get(List<Get> gets) throws IOException`

This is used as follows:

```
List<Get> gets = new ArrayList<Get>();
// Some code to add gets to the list
Result[] results = table.get(gets);
// More code
```

Errors are reported back to you differently than with the Puts. The `get()` call either returns the said array, matching the
same size as the given list by the gets parameter, or throws an exception and not returning a result at all.

#### Delete Method

The _Table_ class provides you with a single and multiple delete options.

##### Single Deletes

The method signature for a single delete is:

```
void delete(Delete delete) throws IOException
```

To instantiate a _Delete_ there are several options:

```
Delete(byte[] row)
Delete(byte[] row, long timestamp)
// The following two allows you to pass in a larger array, with accompanying offset and length parameter
Delete(final byte[] rowArray, final int rowOffset, final int rowLength) 
Delete(final byte[] rowArray, final int rowOffset, final int rowLength,long ts)
Delete(final Delete d)
```

To narrow down what you want to delete, use the following methods:

```
// removes an entire column family, including all contained columns.
Delete addFamily(final byte[] family)
Delete addFamily(final byte[] family, final long timestamp) // narrowed  on timestamp
Delete addFamilyVersion(final byte[] family, final long timestamp)
Delete addColumns(final byte[] family, final byte[] qualifier) // Operates in columns
Delete addColumns(final byte[] family, final byte[] qualifier, final long timestamp) // narrowed on timestamp
Delete addColumn(final byte[] family, final byte[] qualifier) // narrow to a single column (newest)
Delete addColumn(byte[] family, byte[] qualifier, long timestamp) // narrowed by timestamp
void setTimestamp(long timestamp) // Set timestamp for all subsequent addXXX(...) calls
```

The handling of the explicit versus implicit timestamps is the same for all `addXYZ()` methods, if not specified, the
optional one from the constructor or a previous call to `setTimestamp()` is used. If that was not set either, then
_HConstants.LATEST_TIME STAMP_ is used, meaning all versions will be affected by the delete. Not setting the timestamp forces
the server to retrieve the latest timestamp on the server side on your behalf. This is slower than performing a delete with
an explicit timestamp. For advanced user there is an additional method
available `Delete addDeleteMarker(Cell kv) throws IOException`, which checks that the provided _Cell_ instance is of type
delete, and that the row key matches the one of the current delete instance. If that holds true, the cell is added as-is to
the family it came from. The _Delete_ class has other methods too:

    * cellScanner(): Provides a scanner over all cells available in this instance.
    * getACL()/setACL(): The ACLs for this operation (might be null).
    * getAttribute()/setAttribute(): Set and get arbitrary attributes associated with this instance of Delete
    * getAttributesMap(): Returns the entire map of attributes, if any are set
    * getCellVisibility()/setCellVisibility(): The cell level visibility for all included cells
    * getClusterIds()/setClusterIds(): The cluster IDs as needed for replication purposes
    * getDurability()/setDurability(): The durability settings for the mutation
    * getFamilyCellMap()/setFamilyCellMap(): The list of all cells of this instance
    * getFingerprint(): Compiles details about the instance into a map for debugging, or logging 
    * getId()/setId(): An ID for the operation, useful for identifying the origin of a request later 
    * getRow(): Returns the row key as specified when creating the Delete instance 
    * getTimeStamp(): Retrieves the associated timestamp of the Delete instance 
    * getTTL()/setTTL(): Not supported by Delete, will throw an exception when setTTL() is called 
    * heapSize(): Computes the heap space required for the current Delete instance including data and internal structures
    * isEmpty(): Checks if the family map contains any Cell instances
    * numFamilies(): Returns the number of Cell instances that will be
    * size(): applied with this Delete 
    * toJSON()/toJSON(int): Converts the first 5 or N columns into a JSON format
    * toMap()/toMap(int): Converts the first 5 or N columns into a map. More detailed than what getFingerprint() returns 
    * toString()/toString(int): Converts the first 5 or N columns into a JSON, or map (if JSON fails due to encoding)

##### List of Deletes

In the _Table_ class there is a method `void delete(List<Delete> deletes) throws IOException` similar to the Put list seing
before. Just as with the other list-based operation, you cannot make any assumption regarding the order in which the deletes
are applied on the remote servers. The API is free to reorder them to make efficient use of the single RPC per affected
region server. Regarding the error handling, the list-based `delete()` call is modified to only contain the failed delete
instances when the call returns. In other words, when everything has succeeded, the list will be empty. The call also throws
the exception—if there was one—reported from the remote servers (for example if a wrong familly name is passed) with an
exception of type _RetriesExhaustedWithDetailsException_.

##### Atomic Check-and-Delete

Delete also comes with server-side read-modify-write functionality availale in the _Table_ class:

```
boolean checkAndDelete(byte[] row, byte[] family, byte[] qualifier, byte[] value, Delete delete) throws IOException
boolean checkAndDelete(
    byte[] row, 
    byte[] family, 
    byte[] qualifier, 
    CompareFilter.CompareOp compareOp, 
    byte[] value, 
    Delete delete) throws IOException
```

You need to specify the row key, column family, qualifier, and value to check before the delete operation is performed. The
first call implies that the given value has to equal to the stored one. The second call lets you specify the actual
comparison operator. If the test fails, nothing is deleted and the call returns false. Using null as the value parameter
triggers the nonexistence test, which is successful if the column specified does not exist. You can only perform the check
and modify on the same row, if you pass a _Delete_ instance pointing to a different row, the operation will fail.

#### Append Method

The `append()` method does an atomic read-modify-write operation, adding data to a column. The API method provided is:

`Result append(final Append append) throws IOException`

Constructors:

```
Append(byte[] row)
Append(final byte[] rowArray, final int rowOffset, final int rowLength) // subset, plus the necessary offset and length
Append(Append a) // clone Append
```

Once the instance is created, you add details of the column you want to append to, using one of these calls:

```
Append add(byte[] family, byte[] qualifier, byte[] value) // column family, qualifier name and value to add to the existing 
Append add(final Cell cell) // copies all of these parameters from an existing cell instance
```

You must call one of those functions, or else a subsequent call to `append()` will throw an exception. One special option
of `append()` is to not return any data from the servers. This is accomplished with this pair of methods (usually, the newly
updated cells are returned to the caller):

```
Append setReturnResults(boolean returnResults) // will return null instead
boolean isReturnResults()
```

Most of the methods of the _Append_ class comes from the superclasses and have been mentioned before.

#### Mutate Method

##### Single Mutations

If you want to update a row across operations, and doing so atomically, you can use `mutateRow()`, which has the following
signature `void mutateRow(final RowMutations rm) throws IOException`. The _RowMutations_ based parameter is a container that
accepts either _Put_ or _Delete_ instance, and then applies both in one call to the server-side data. The list of available
constructors and methods for the RowMutations class is:

```
RowMutations(byte[] row)
add(Delete)
add(Put)
getMutations()
getRow()
```

An example of usage:

```
Put put = new Put(Bytes.toBytes("row1"));
put.addColumn(Bytes.toBytes("colfam1"), Bytes.toBytes("qual1"), 4, Bytes.toBytes("val99")); // adds value
put.addColumn(Bytes.toBytes("colfam1"), Bytes.toBytes("qual4"),4, Bytes.toBytes("val100")); // add new familly

Delete delete = new Delete(Bytes.toBytes("row1")); 
delete.addColumn(Bytes.toBytes("colfam1"), Bytes.toBytes("qual2")); // Delete column familly

RowMutations mutations = new RowMutations(Bytes.toBytes("row1")); 
mutations.add(put);
mutations.add(delete);
table.mutateRow(mutations); // perform the operations atomically
```

##### Atomic Check-and-Mutate

Again, equivalent calls for mutations that give you access to server-side read-modify-write functionality:

```
// You need to specify the row key, column family, qualifier, and value to check before the list of mutations is applied
// If test fail, nothing is applied and the call returns a false, otherwise it returns true.
public boolean checkAndMutate(
        final byte[] row, 
        final byte[] family, 
        final byte[] qualifier, 
        final CompareOp compareOp, 
        final byte[] value, 
        final RowMutations rm) throws IOException
```

Again, using null as the value parameter triggers the non‐existence test, that is, the check is successful if the column
specified does not exist. You can only perform the check-and-modify operation on the same row or else it will throw an
exception.

### Batch Operations

Now we will have a look at the API calls to batch different operations across multiple rows. As a note, avoid the previously
described list calls and use the batch counterpart instead. The following methods on the _Table_ class are batch:

```
// Row is a superclass of all Mutation classes
void batch(final List<? extends Row> actions, final Object[] results) throws IOException, InterruptedException
void batchCallback(final List<? extends Row> actions, final Object[] results, final Batch.Callback<R> callback) 
    throws IOException, InterruptedException
```

The general contract of the batch calls is that they return a best match result per input action, and the possible types are:

    * null: The operation has failed to communicate with the remote server 
    * Empty Result: Returned for successful Put and Delete operations
    * Result: Returned for successful Get operations, but may also be empty when there was no matching row or column
    * Throwable: In case the servers return an exception for the operation it is returned to the client as-is

All the operations are grouped by the destination region servers first and then sent to the servers. Also, all batch
operations are executed before the results are checked: even if you receive an error for one of the actions, all the other
ones have been applied.

### Scans

#### Introduction

Scans are similar to cursors in database systems, similar to the _Get_ operations. Since scans are similar to iterators,
there is no `scan()` call, but rather a `getScanner()` which returns a _ResultScanner_ instance that you can iterate over.
The available methods are:

```
ResultScanner getScanner(Scan scan) throws IOException
ResultScanner getScanner(byte[] family) throws IOException 
ResultScanner getScanner(byte[] family, byte[] qualifier) throws IOException
```

The Scan class has the following constructors:

```
Scan()
Scan(byte[] startRow, Filter filter)
Scan(byte[] startRow)
Scan(byte[] startRow, byte[] stopRow)
Scan(Scan scan) throws IOException
Scan(Get get)
```

Instead of specifying a single row key, you now can optionally provide a _startRow_ (inclusive) parameter—defining the row
key where the scan begins to read from the HBase table. The optional _stopRow_ (exclusive) parameter can be used to limit the
scan to a specific row key where it should conclude the reading. With scans, you do not need to have an exact match for
either of these rows. Instead, the scan will match the first row key that is equal to or larger than the given start row. If
no start row is provided, it will start from the beginning of the table. It will also end its work when the current row key
is equal to or greater than the optional stop row. If no stop row was specified, the scan will run to the end of the table.
The get and scan functionality is the same on the server side, but for a _Get_ the scan has to include the stop row into the
scan, since both, the start and stop row are set to the same value. The _Get_ instance would return true as well when the
method `boolean isGetScan()` is called. To narrow down the scan results, you can use one of the following methods:

```
Scan addFamily(byte [] family)
Scan addColumn(byte[] family, byte[] qualifier)
```

Scan has other methods that are selective in nature, here the first set that center around the cell versions returned:

```
Scan setTimeStamp(long timestamp) throws IOException // shorthand for setting a time range with setTimeRange(time, time + 1)
Scan setTimeRange(long minStamp, long maxStamp) throws IOException
TimeRange getTimeRange()
Scan setMaxVersions()
Scan setMaxVersions(int maxVersions)
int getMaxVersions()
```

The next set of methods relate to the rows that are included in the scan:

```
Scan setStartRow(byte[] startRow)
byte[] getStartRow() // might return null
Scan setStopRow(byte[] stopRow)
byte[] getStopRow() // might return null

// Shorthand to set the start row to the value of the rowPrefix parameter and the stop row to the next key that is 
// greater than the current key
Scan setRowPrefixFilter(byte[] rowPrefix)
```

Next, there are methods around filters (combination of time range and row based selectors):

```
Filter getFilter()
Scan setFilter(Filter filter)
boolean hasFilter()
```

And finally more specific methods for advanced usage:

```
Scan setReversed(boolean reversed) // Iterate the scan in reverse order if set to true, slower (descending order)
boolean isReversed() // returns if the scan is reversed
Scan setRaw(boolean raw) // Special mode, the scan returns every cell it finds, including deleted
boolean isRaw()
Scan setSmall(boolean small) // Deals with scans that read a very small set of data which can be returned in a single RPC
boolean isSmall()
```

When using reverse scans you need to flip around any start and stop row value, or you will not find anything at all. Calling
`setSmall(true)` on a scan instance instructs the client API to not do the usual open scanner, fetch data, and close scanner
combination of remote procedure calls, but do them in one single call (size should ideally be less than one data block).

#### The ResultScanner Class

Scans usually do not ship all the matching rows in one RPC to the client, but instead do this on a per-row basis, the
_ResultScanner_ converts the scan into a get-like operation, wrapping the Result instance for each row into an iterator:

```
Result next() throws IOException // Returns null if you exhaust the table
Result[] next(int nbRows) throws IOException // The result array might be shorter if there were not enough rows left
void close() // Required to release all the resources a scan may hold 
```

Release a scanner instance as quickly as possible as an open scanner holds quite a few resources on the server side. This
time can be configured with the following property in the hbase-site.xml:

```xml

<property>
    <name>hbase.client.scanner.timeout.period</name>
    <value>120000</value>
</property>
```

#### Scanner Caching

Each call to `next()` or `next(int nbRows)` is a separate RPC for every row, but sometimes it makes sense to fetch more than
one row per RPC if possible. This is called _scanner caching_ and is enabled by default. _hbase.client.scanner.caching_
controls the default caching for all scans (defaults to 100). This can be overwritten at the _Scan_ class with the methods:

```
void setCaching(int caching)
int getCaching()
```

Values to set here should be chosen carefully, because when the time taken to transfer the rows to the client, or to process
the data on the client, exceeds the configured scanner lease threshold, you will end up receiving a lease expired error, in
the form of a _ScannerTimeoutException_ being thrown. If you want to change the lease period setting you need to add the
appropriate configuration key to the hbase-site.xml file on the region servers.

#### Scanner Batching

The previous technique of caching doesn't work well when the rows to retrieve are huge, but in this situations you can use
batching, which is configurable at the _Scan_ class:

```
void setBatch(int batch)
int getBatch()
```

Batching works at cell level, and controls how many cells are retrieved for every call to `next()` made. When a row contains
more cells than the value you used for the batch, you will get the entire row piece by piece, with each next Result returned
by the scanner. The combination of scanner caching and batch size can be used to control the number of RPCs required to scan
the row key range selected.

Batching cannot be combined with filters that return true from their `hasFilterRow()` method as filters can not deal with
partial results, this is, the row being chunked into batches. It needs to see the entire row to make a filtering decision.
Another combination disallowed is batching with small scans. If you try to set the scan batching and small scan flag
together, you will receive an _IllegalArgumentException_.

#### Slicing Rows

The following methods are available on the _Scan_ class to slice a table:

```
Scan setMaxResultsPerColumnFamily(int limit) // max results per column family limit to stop returning data once reached
int getMaxResultsPerColumnFamily()
Scan setRowOffsetPerColumnFamily(int offset) // offset (number of cells) to start from a specific column
int getRowOffsetPerColumnFamily()
Scan setMaxResultSize(long maxResultSize) // sets an upper size limit in bytes to the data returned by the scan
long getMaxResultSize()
```

#### Load Column Families on Demand

Scans have another advanced feature, loading column families on demand. This is controlled by the following methods:

```
Scan setLoadColumnFamiliesOnDemand(boolean value)
Boolean getLoadColumnFamiliesOnDemandValue()
boolean doLoadColumnFamiliesOnDemand()
```

This functionality is a read optimization, useful then for use-cases with a dependency between data in those families. This
is achieved through calling `setLoadColumnFamiliesOnDemand(true)` and then passing a filter that implements the following
method, returning a boolean flag `boolean isFamilyEssential(byte[] name) throws IOException`. When the servers scan the data,
they first set up internal scanners for each column family. If load column families on demand is enabled and a filter set, it
calls out to the filter and asks it to decide if an included column family is to be scanned or not.

#### Scanner Metrics

The _Scan_ class provides functionality to access metrics on the operation through the following methods:

```
Scan setScanMetricsEnabled(final boolean enabled)
boolean isScanMetricsEnabled()
ScanMetrics getScanMetrics()
```

For example, to find out the number of RPCs calls used in a _Scan_, you can access the metrics instance:

```
Scan scan = new Scan().setCatching(....
ResultScanner scanner = table.getScanner(scan);
scanner.close();
...
ScanMetrics metrics = scan.getScanMetrics();
System.out.println("RPCs: " + metrics.countOfRPCcalls);
```

The returned ScanMetrics instance has a set of fields and methods to access the metrics:

    * countOfRPCcalls: The total amount of RPC calls incurred by the scan.
    * countOfRemoteRPCcalls: The amount of RPC calls to a remote host.
    * sumOfMillisSecBetweenNexts: The sum of milliseconds between sequential next() calls.
    * countOfNSRE: Number of NotServingRegionException caught.
    * countOfBytesInResults: Number of bytes in Result instances returned by the servers.
    * countOfBytesInRemoteResults: Same as above, but for bytes transferred from remote servers.
    * countOfRegions: Number of regions that were involved in the scan.
    * countOfRPCRetries: Number of RPC retries incurred during the scan
    * countOfRemoteRPCRetries: Same again, but RPC retries for non-local servers.

Metrics are internally collected region by region and accumulated in the _ScanMetrics_ instance of the _Scan_ object.

### Miscellaneous Features

#### The Table Utility Methods

Other available methods on the _Table_ class:

    * void close(): Call the method on work completion to release the resources held
    * TableName getName(): Convenience method to retrieve the table name and namespace, provided as a TableName instance
    * Configuration getConfiguration(): allows you to access the configuration in use by the Table instance. Since this 
      is handed out by reference, you can make changes that are effective immediately
    * HTableDescriptor getTableDescriptor(): Each table is defined using an instance of the HTableDescriptor class

#### The Bytes Class

Class used to convert native Java types, such as String, or long, into the raw, byte array format HBase supports natively.
Most methods come in three variations, for example shown here for the _Long_ type:

```
static long toLong(byte[] bytes)
static long toLong(byte[] bytes, int offset)
static long toLong(byte[] bytes, int offset, int length) // length has to match the length of the native type
```

The API, and HBase internally, store data in larger arrays using, for example, the following call:
`static int putLong(byte[] bytes, int offset, long val)`

Other methods that are worthy to know:

    * toStringBinary(): converts non-printable data into human-readable hexadecimal numbers
    * compareTo()/equals(): allow you to compare two byte arrays, returns a comparison value or a boolean respectively
    * add()/head()/tail(): use these methods to combine two byte arrays
    * binarySearch(): performs a binary search in the given array of values
    * incrementBytes(): increments a long value in its byte array representation, as if you used toBytes(long) to create it

## Chapter 4: Client API: Advanced Features<a name="Chapter4"></a>

### Filters

#### Introduction to Filters

You can limit the data retrieved in the _Get_ or _Scan_ operations by passing a _Filter_, used for features such as selection
of keys, or values, based on regular expressions. The _Filter_ interface is implemented by some provided HBase classes or you
can define your own, Filters are applied at server side (predicate pushdown).

##### The Filter Hierarchy

The base interface as said before is _Filter_, the abstract class _FilterBase_ contains boilerplate for some common
functionality and most concrete implementations of the _Filter_ extends from it. Set the filter in the _Get_ or _Scan_
classes by using the method `setFilter(Filter filter)`. Some implementations of _Filter_ requires some extra parameters on
instantiation, like the ones extending _CompareFilter_ which asks for two parameters.

Filters have access to the entire row they are applied to, as they might depend on row key, column qualifiers, actual value
of a column, timestamps, and so on. Filters have no state and cannot span across multiple rows.

##### Comparison Operators

The possible comparison operators for CompareFilter-based filters are listed below:

    * LESS: Match values less than the provided one.
    * LESS_OR_EQUAL: Match values less than or equal to the provided one.
    * EQUAL: Do an exact match on the value and the provided one.
    * NOT_EQUAL: Include everything that does not match the provided value.
    * GREATER_OR_EQUAL: Match values that are equal to or greater than the provided one.
    * GREATER: Only include values greater than the provided one.
    * NO_OP: Exclude everything.

##### Comparators

The second type that you need to provide to _CompareFilter_ classes is a comparator, which is needed to compare various
values and keys in different ways. These are derived from _ByteArrayComparable_, which implements the Java
_Comparable_ interface. Some specific examples of these are listed below:

    * LongComparator: Assumes the given value array is a Java Long number and uses Bytes.toLong() to convert it
    * BinaryComparator: Uses Bytes.compareTo() to compare the current with the provided value
    * BinaryPrefixComparator: Similar to the above, but does a left hand, prefix-based match using Bytes.compareTo()
    * NullComparator: Does not compare against an actual value, but checks whether a given one is null, or not null
    * BitComparator: Performs a bitwise comparison, providing a BitwiseOp enumeration with AND, OR, and XOR operators
    * RegexStringComparator: Given a regular expression at instantiation, this comparator does a pattern match on the data
    * SubstringComparator: Treats the value and table data as String instances and performs a contains() check

Some of these constructors take a byte array, to do the binary comparison, while others take a string parameter. The later
are computationally more expensive.

#### Comparison Filters

The comparison filters takes the comparison operator and comparator instance as described before. The constructor is:  
`CompareFilter(final CompareOp compareOp, final ByteArrayComparable comparator)`
Filters based on _CompareFilter_ are doing the opposite than normal filters, which is, defining which elements include in the
result rather to which ones to exclude.

##### RowFilter

Filter that gives you the ability to filter data based on row keys. Example of usage:

```
Scan scan = new Scan();
scan.addColumn(Bytes.toBytes("column_family1"), Bytes.toBytes("column1"));
// Returns rows using the binary comparator, returns for example row09, row08...
Filter filter1 = new RowFilter(CompareFilter.CompareOp.LESS_OR_EQUAL, new BinaryComparator(Bytes.toBytes("row10")));
// Exact match needed to match the regex, for example row-15, row-25...
Filter filter2 = new RowFilter(CompareFilter.CompareOp.EQUAL,new RegexStringComparator(".*-.5"));
// Substring comparator,  returns rows that contains '-5', for example row-5, row-50...
Filter filter3 = new RowFilter(CompareFilter.CompareOp.EQUAL,new SubstringComparator("-5"));

scan.setFilter(filter1);
ResultScanner scanner = table.getScanner(scan);
for (Result res : scanner) {
    System.out.println(res);
}
scanner.close();
```

##### FamilyFilter

Similar to the _RowFilter_, but applies the comparison to the column families available in a row. For example:
`new FamilyFilter(CompareFilter.CompareOp.LESS, new BinaryComparator(Bytes.toBytes("column_family")))`
Which filters the columns belonging to the family 'column_family'.

##### QualifierFilter

Similar to the _FamilyFilter_ but filters on column qualifier instead:
`new QualifierFilter(CompareFilter.CompareOp.LESS_OR_EQUAL, new BinaryComparator(Bytes.toBytes("col-2")))`
The above example would return columns with names like 'col-1','col-10'... As the comparison is done lexicographically.

##### ValueFilter

Filter that makes it possible to include only cells that have a specific value. For example:
`new ValueFilter(CompareFilter.CompareOp.EQUAL, new SubstringComparator(".4"))`
The above example returns cells with values like 'val-1.4','otherval-2.4','yetanotherval-0.4'...

##### DependentColumnFilter

This filter lets you specify a dependent column, or reference column, that controls how other columns are filtered. It uses
the timestamp of the reference column and includes all other columns that have the same timestamp. The constructors are:

```
DependentColumnFilter(final byte[] family, final byte[] qualifier)
DependentColumnFilter(final byte[] family, final byte[] qualifier, final boolean dropDependentColumn)
DependentColumnFilter(final byte[] family, final byte[] qualifier, final boolean dropDependentColumn, final CompareOp
    valueCompareOp, final ByteArrayComparable valueComparator)
```

Think of it as a combination of a _ValueFilter_ and a filter selecting on a reference timestamp. This filter is not
compatible with the batch feature of the scan operations, that is, setting `Scan.setBatch()` to a number larger than zero.
The filter needs to see the entire row to do its work, and using batching will not carry the reference column timestamp over
and would result in erroneous results. This filter is used where applications require client-side timestamps to track
dependent updates.

#### Dedicated Filters

The second type of supplied filters are based directly on _FilterBase_, many of these filters are only really applicable when
performing scan operations, since they filter out entire rows.

##### PrefixFilter

All rows with a row key matching this prefix are returned to the client. The constructor is:
`PrefixFilter(final byte[] prefix)`
The scan also is actively ended when the filter encounters a row key that is larger than the prefix. Combining this with a
start row improves the overall performance of the scan as it has knowledge of when to skip the rest of the rows altogether.

##### PageFilter

When you create the instance, you specify a _pageSize_ parameter, which controls how many rows per page should be returned.
`PageFilter(final long pageSize)`
There is a fundamental issue with filtering on physically separate servers, as filters runs on different region servers in
parallel and cannot retain or communicate their current state across those boundaries. Each filter is required to scan at
least up to _pageCount_ rows before ending the scan. This means a slight inefficiency is given for the Page Filter as more
rows are reported to the client than necessary. The client code would need to remember the last row that was returned, and
then, when another iteration is about to start, set the start row of the scan accordingly. Example of usage:

```
private static final byte[] POSTFIX = new byte[] { 0x00 };
Filter filter = new PageFilter(15);
int totalRows = 0;
byte[] lastRow = null;

while (true) {
    Scan scan = new Scan();
    scan.setFilter(filter);
    if (lastRow != null) {
        byte[] startRow = Bytes.add(lastRow, POSTFIX);
        scan.setStartRow(startRow);
    }
    ResultScanner scanner = table.getScanner(scan);
    Result result;
    while ((result = scanner.next()) != null) {
        totalRows++;
        lastRow = result.getRow();
    }
    scanner.close();
}
```

Because of the lexicographical sorting of the row keys by HBase and the comparison taking care of finding the row keys in
order, and the fact that the start key on a scan is always inclusive, you need to add an extra zero byte (smallest increment)
to the previous key. This will ensure that the last seen row key is skipped and the next, in sorting order, is found.

##### KeyOnlyFilter

This filter returns just the keys of each Cell, while omitting the actual data. The constructors of the filter are:
`KeyOnlyFilter()` and `KeyOnlyFilter(boolean lenAsVal)`, when the latter is used and passed as true, the value returned is
the length (in bytes) of the data.

##### FirstKeyOnlyFilter

Filter that allows you to get the first column of each row. Typically used by row counter type applications that only need to
check if a row exists ( if there are no columns, the row ceases to exist). Another possible use case is to set the column
qualifier to an epoch value, so this would sort the column with the oldest timestamp name as the first to retrieve. Combined
with this filter, it is possible to retrieve the oldest (or newest if reversed) column from every row using a single scan.
Another optimization feature provided by the filter is that it indicates to the region server applying the filter that the
current row is done and that it should skip to the next one.

##### FirstKeyValueMatchingQualifiersFilter

Extension to the _FirstKeyOnlyFilter_, but instead of returning the first found cell, it instead returns all the columns of a
row, up to a given column qualifier (if it has it, otherwise returns all columns). The constructor is
`FirstKeyValueMatchingQualifiersFilter(Set<byte[]> qualifiers)`
Note that you can pass several qualifiers, the filter is instructed to stop emitting cells when encountering one of the
qualifiers provided.

##### InclusiveStopFilter

The row boundaries of a scan are inclusive for the start row and exclusive for the stop row. This filter includes also the
specified stop row.

##### FuzzyRowFilter

This filter acts on row keys, but in a fuzzy manner. It needs a list of row keys that should be returned, plus an
accompanying byte[] array that signifies the importance of each byte in the row key:
`FuzzyRowFilter(List<Pair<byte[], byte[]>> fuzzyKeysData)`
The _fuzzyKeysData_ specifies the mentioned significance of a row key byte, by taking one of two values:

    * 0: Indicates that the byte at the same position in the row key must match as-is
    * 1: Means that the corresponding row key byte does not matter and is always accepted

For example, the row key to find any row with the format `<year_4_digits>_<month_2_digits>_<day_2_digits>` that matches any
day of february 2021 would be `2021_02_??` and the _fuzzyKeysData_ value is `\x01\x01\x01\x01\x01\x01\x01\x01\x00\x00`. An
advantage of this filter is that it can likely compute the next matching row key when it gets to an end of a matching one. It
implements the _getNextCellHint()_ method to help the servers in fast-forwarding to the next range of rows that might match.
This skips scanning blocks and speeds up the scan operations.

##### ColumnCountGetFilter

This filter retrieves a specific maximum number of columns per row. Set the number using the constructor of the filter:
`ColumnCountGetFilter(final int n)`
This filter stops the entire scan once a row has been found that matches the maximum number of columns configured.

##### ColumnPaginationFilter

Similar to the PageFilter, this one can be used to page through columns in a row. Its constructor has two parameters:
`ColumnPaginationFilter(final int limit, final int offset)`
It skips all columns up to the number given as offset, and then includes limit columns afterward (superseded by slicing).

##### ColumnPrefixFilter

Similar to the previous one, but acts on columns, the constructor is `ColumnPrefixFilter(final byte[] prefix)`. All columns
that have the given prefix are then included in the result.

##### MultipleColumnPrefixFilter

Extension to the ColumnPrefixFilter, allows the application to ask for a list of column qualifier prefixes, instead of one.
`MultipleColumnPrefixFilter(final byte[][] prefixes)`

##### ColumnRangeFilter

This filter acts like two QualifierFilter instances working together, with one checking the lower boundary, and the other
doing the same for the upper. The constructor is:
`ColumnRangeFilter(final byte[] minColumn, boolean minColumnInclusive, final byte[] maxColumn, boolean maxColumnInclusive)`
If you don't pass a min/max column, the first/last column would be used.

##### SingleColumnValueFilter

Use this filter when you have exactly one column that decides if an entire row should be returned or not. You need to first
specify the column you want to track, and then some value to check against. The constructors offered are:

```
// Creates a Binary Comparator instance internally on your behalf
SingleColumnValueFilter(
    final byte[] family,
    final byte[] qualifier,
    final CompareOp compareOp,
    final byte[] value) 
// Same parameters we used for the CompareFilter-based classes
SingleColumnValueFilter(
    final byte[] family, 
    final byte[] qualifier, 
    final CompareOp compareOp, 
    final ByteArrayComparable comparator)
// The boolean flags here can also be set by using the class setters after constructing it. The filterIfMissing controls 
// if rows with no columns after filtering should appear in the results, the second is self-explanatory
protected SingleColumnValueFilter(
    final byte[] family, 
    final byte[] qualifier,
    final CompareOp compareOp, 
    ByteArrayComparable comparator, 
    final boolean filterIfMissing, 
    final boolean latestVersionOnly)
```

##### SingleColumnValueExcludeFilter

Extension of the previous one, but the reference column is omitted from the results

##### TimestampsFilter

When you need fine-grained control over what versions are included in the scan result. The constructor is:
`TimestampsFilter(List<Long> timestamps)`
The filter is asking for a list of timestamps, it will attempt to retrieve the column versions with the matching timestamps.

##### RandomRowFilter

Filter that includes random rows in the results. The constructor is `RandomRowFilter(float chance)`, where change is a number
between 0 and 1, a negative value will exclude all rows, while a number greater than 1 will include all.

#### Decorating Filters

It is sometimes useful to modify, or extend, the behavior of a filter to gain additional control over the returned data.

##### SkipFilter

This filter wraps a given filter (must implement `filterKeyValue`) and extends it to exclude an entire row, when the wrapped
filter hints for a _Cell_ to be skipped. As soon as the underlying filter hints a cell to be omitted, the entire row is
omitted too. Example below:

```
Filter filter = new ValueFilter(CompareFilter.CompareOp.NOT_EQUAL, new BinaryComparator(Bytes.toBytes("val-0")));
Scan scan = new Scan();
scan.setFilter(new SkipFilter(filter));
ResultScanner scanner1 = table.getScanner(scan);
// Other code
```

##### WhileMatchFilter

Filter that adds the behaviour of aborting the entire scan once a piece of information is filtered.

#### FilterList

_FilterList_ is used when you want to have more than a filter being applied to reduce the data returned to your application.
The following constructors are available:

```
FilterList(final List<Filter> rowFilters) // row filters are the filters passed together
FilterList(final Filter... rowFilters)
FilterList(final Operator operator) // operator is used to combine the results, defaults to MUST_PASS_ALL
FilterList(final Operator operator, final List<Filter> rowFilters)
FilterList(final Operator operator, final Filter... rowFilters)
```

The possible values for the _Operator_ above are:

    * MUST_PASS_ALL: A value is only included in the result when all filters agree to do so
    * MUST_PASS_ONE: As soon as a value was allowed to pass one of the filters, it is included in the overall result

Adding filters, after the FilterList instance has been created, can be done with `void addFilter(Filter filter)`. You are
free to add other FilterList instances to an existing FilterList, thus creating a hierarchy of filters, combined with the
operators you need. Choose the List type carefully to control the execution of the filters (i.e ordered list).

#### Custom Filters

To implement your own filter, either implementing the abstract _Filter_ class, or extend the provided _FilterBase_ class.
The _Filter_ interface has an enumeration _ReturnCode_ to be returned by the `filterKeyValue()` method, which values are:

    * INCLUDE: Include the given Cell instance in the result
    * INCLUDE_AND_NEXT_COL: Include current cell and move to next column, i.e. skip all further versions of the current 
    * SKIP: Skip the current cell and proceed to the next
    * NEXT_COL: Skip the remainder of the current column, proceeding to the next. This is used by the TimestampsFilter 
    * NEXT_ROW: Similar to the previous, but skips the remainder of the current row, moving to the next. Used by RowFilter
    * SEEK_NEXT_USING_HINT: Some filters want to skip a variable number of cells and use this return code to indicate 
      that the framework should use the getNextCellHint() method to determine where to skip to. Used by ColumnPrefixFilter 

Most of the provided method of _Filter_ are executed in order, the sequence will be:

    1 - hasFilterRow(): This method does two things: decide if the filter is clashing with other read settings, and call 
      the filterRow() and filter RowCells() methods subsequently. Forces to load the whole row before calling these methods
    2 - filterRowKey(byte[] buffer, int offset, int length): Checks the row key. Use it to skip an entire row from processing
    3 - filterKeyValue(final Cell v): When a row is not filtered (yet), the framework invokes this method for every Cell 
      that is part of the current row being materialized for the read. The ReturnCode indicates what to do with the cell
    4 - transfromCell(): Once the cell has passed the check and is available, the transform call allows the filter to 
      modify the cell, before it is added to the resulting row
    5 - filterRowCells(List<Cell> kvs): Once all row and cell checks have been performed, this method is called, giving you 
      access to the list of Cell instances not excluded by the previous filter methods. The _DependentColumnFilter uses it
    6 - filterRow():After everything else was checked and invoked, the final inspection is performed using filterRow()
      is reached, returning true afterward. The default false would include the current row in the result
    7 - reset(): Resets the filter for every new row the scan is iterating over. Called by the server, after a row is read
    8 - filterAllRemaining(): This method can be used to stop the scan, by returning true. Used by filters for optimization

A filter using _filterRow()_ to filter out an entire row, or filter _RowCells()_ to modify the final list of included cells,
must also override the _hasRowFilter()_ function to return true. Additional methods on _Filter_;

    * getNextCellHint(): Invoked when the filter’s filterKeyValue() method returns ReturnCode.SEEK_NEXT_USING_HINT. 
      Use it to skip large ranges of rows—if possible
    * isFamilyEssential(): Used to avoid unnecessary loading of cells from column families in low-cardinality scans
    * setReversed()/isReversed(): Indicates the filter direction. A reverse scan must use reverse filters too
    * toByteArray()/parseFrom(): Used to de-/serialize the filter’s internal state to ship to the servers for application

##### Custom Filter Loading

To use your custom filter, package it in a JAR, and make it available in the region servers. Then there are 2 choices to load
them:

    * Static Configuration: add the JAR file to the hbase-env.sh, make sure it is loaded in the HBASE_CLASSPATH property
    * Dynamic Loading: Place the jar in the shared JAR file directory in HDFS defined in the hbase.dynamic.jars.dirproperty 
      on the hbase-defaults.xml file, which usually points to hdfs://<namenode>/hbase/lib/ 
      The dynamic loading directory is monitored for changes and will refresh the JAR files locally if they have been updated
      HBase is currently not able to unload a previously loaded class, so you cannot replace a class with the same name

#### Filter Parser Utility

_ParseFilter_ is a helper class used in all the places where filters need to be described with text and then converted to a
Java class, like gateway servers, or the _hbase-shell_ like in:
`hbase(main):001:0> scan 'testtable', { FILTER => "SKIP PrefixFilter('row-2') AND QualifierFilter(<=,'binary:col-2')" }`
In the above example, the "binary:col-2" parameter is divided in the value handed into the filter 'col-2', and the part
'binary' which is the way the filter parser class allows you to specify a comparator for filters based on _CompareFilter_.
Other allowed comparators are:

    * binary: BinaryComparator 
    * binaryprefix: BinaryPrefixComparator
    * regexstring: RegexStringComparator
    * substring: SubstringComparator

Also, the following comparison operators are allowed:

    * `<`: CompareOp.LESS
    * `<=`: CompareOp.LESS_OR_EQUAL
    * `>`: CompareOp.GREATER
    * `>=`: CompareOp.GREATER_OR_EQUAL = CompareOp.EQUAL
    * `!=`: CompareOp.NOT_EQUAL

Combiners for the filter and precedence ot it is the following:

    * SKIP/WHILE: Wrap filter into SkipFilter, or WhileMatchFilter instance.
    * AND: corresponds with the MUST_PASS_ALL filter list
    * OR:  corresponds with the MUST_PASS_ONE filter list

The ParseFilter class only supports the filters that are shipped with HBase.

### Counters

#### Introduction to Counters

HBase offers the _Counter_ class to deal with statistics. HBase has a mechanism to treat columns as counters instead of the
lock, read, modify, unlock mechanism required otherwise which has as a lot of contention. The client API provides specialized
methods to do the read-modify-write operation atomically in a single client-side call. Example on how to operate on counters:

```

hbase(main):001:0> create 'counters', 'daily', 'weekly', 'monthly'
hbase(main):002:0> incr 'counters', '20150101', 'daily:hits', 1 // Increses the counter by 1 -> COUNTER VALUE = 1
hbase(main):003:0> incr 'counters', '20150101', 'daily:hits', 1 // Increses the counter by 1 -> COUNTER VALUE = 2
hbase(main):04:0> get_counter 'counters', '20150101', 'daily:hits' // COUNTER VALUE = 2
```

The `incr` operation is called with `incr '<table>', '<row>', '<column>', [<increment-value>]`. Counters require no
initialization (set to 0) and by default are incremented by 1 if the increment value is not passed. Counters can
programmatically be read using `Bytes.toLong()` and `Bytes.toBytes(long)` to encode the value (make sure it is a long). You
can also read a counter with `hbase(main):005:0> get 'counters', '20150101'` but the result is given in bytes and thus not
readable, or use `hbase(main):007:0> get_counter 'counters', '20150101', 'daily:hits'` to get it in a readable format.

#### Single Counters

The first type of increment call is for single counters only. The methods, provided by _Table_ are:

```
long incrementColumnValue(byte[] row, byte[] family, byte[] qualifier, long amount) throws IOException
long incrementColumnValue(byte[] row, byte[] family, byte[] qualifier, long amount, Durability durability) throws IOException
```

For example `table.incrementColumnValue(Bytes.toBytes("20110101"),Bytes.toBytes("daily"), Bytes.toBytes("hits"), 1)`

#### Multiple Counters

Another way to increment counters is provided by the `increment()` call of _Table_. The method is
`Result increment(final Increment increment) throws IOException`
You must create an instance of the Increment class and fill it with the appropriate details. The constructors provided are:

```
Increment(byte[] row)
Increment(final byte[] row, final int offset, final int length)
Increment(Increment i)
```

You must provide a row key when instantiating an Increment. There is also the variant for larger arrays with an offset and
length parameter to extract the row key from, and finally the clonning constructor. After instantiation, add the actual
counters you want to increment, using these methods:

```
Increment addColumn(byte[] family, byte[] qualifier, long amount) // Takes the column coordinates
Increment add(Cell cell) throws IOException // reuses an existing cell, useful if you already have hold of the counter
```

Counters as opposite of _Put_, don't have a version or timestamp, nor family. Counters takes an optional time range, which is
passed on to the servers to restrict the internal get operation from retrieving the current counter values:

```
Increment setTimeRange(long minStamp, long maxStamp) throws IOException
TimeRange getTimeRange()
```

The above can be used to expire counters, for example, to partition them by time. Increment gets most of the methods from
the _Mutation_ class, and add the following:

    * Map<byte[], NavigableMap<byte[], Long>> getFamilyMapOfLongs(): Returns a list of Long instance, instead of cells, for 
      what was added to this instance so far. The list is indexed by families, and then by column qualifier

### Coprocessors

With the coprocessor feature (advanced feature), you can move part of the computation to where the data lives.

#### Introduction to Coprocessors

A coprocessor enables you to run arbitrary code directly on each region server. Coprocessors executes code on a per-region
basis, giving you trigger-like functionality similar to stored procedures in the RDBMS world. There is a set of implicit
events that you can use to hook into, or you can also extend the RPC protocol to introduce your own set of calls. You need to
create the java classes and make them available to the region servers, the same way it has been shown for custom filters.
HBase provides classes, based on the coprocessor, which you can use to extend from when implementing your own functionality:

    * Endpoint: Endpoints are dynamic extensions to the RPC protocol, adding callable remote procedures
    * Observer: Comparable to triggers: callback functions are executed when certain events occur. Some implementators:
        - MasterObserver: to react to administrative or DDL-type operations. These are cluster-wide events
        - RegionServerObserver: Hooks into commands sent to a region server, and covers region server-wide events
        - RegionObserver: Used to handle data manipulation events. They are closely bound to the regions of a table
        - WALObserver: This provides hooks into the write-ahead log processing, which is region server-wide
        - BulkLoadObserver: Handles events around the bulk loading API. Triggered before and after the loading takes place
        - EndpointObserver: Whenever an endpoint is invoked by a client, this observer is providing a callback method

Coprocessors can be chained, very similar to what the Java Servlet API does with request filters.

#### The Coprocessor Class Trinity

All user coprocessor classes must be based on the Coprocessor interface. The interface provides two sets of types, which are
used throughout the framework: the _PRIORITY_ constants, and _State_ enumeration. The priority of a coprocessor defines in
what order the coprocessors are executed (system-level instances are called before the user-level coprocessors are executed).
The possible values for this constant are shown below:

    * PRIORITY_HIGHEST: Highest priority, serves as an upper boundary.
    * PRIORITY_SYSTEM: High priority, used for system coprocessors (Integer.MAX_VALUE / 4).
    * PRIORITY_USER: For all user coprocessors, which are executed subsequently (Integer.MAX_VALUE / 2).
    * PRIORITY_LOWEST: Lowest possible priority, serves as a lower boundary (Integer.MAX_VALUE).

Within each priority level, there is also the notion of a sequence number, which keeps track of the order in which the
coprocessors were loaded. the Coprocessor interface offers two calls:

```
void start(CoprocessorEnvironment env) throws IOException
void stop(CoprocessorEnvironment env) throws IOException
```

The provided Coprocessor Environment instance is used to retain the state across the lifespan of the coprocessor instance. A
coprocessor instance is always contained in a provided environment, which provides the following methods:

```
String getHBaseVersion() // Returns the HBase version
int getVersion() // Returns the version of the Coprocessor interface.
Coprocessor getInstance() // Returns the loaded coprocessor instance.
int getPriority() // Provides the priority level of the coprocessor.
int getLoadSequence() // Sequence number of the coprocessor. Set when the instance is loaded and reflects the execution order
Configuration getConfiguration() // Provides access to the current, server-wide configuration.
// Returns a Table implementation for the given table name. Allows the coprocessor to access the actual table data
HTableInterface getTable(TableName tableName) 
// Same than the above but allows the specification of a custom ExecutorService instance.
HTableInterface getTable(TableName tableName, ExecutorService service)
```

Each step in the process has a well known state. The life-cycle state values provided by the coprocessor interface are:

    * UNINSTALLED: The coprocessor is in its initial state. It has no environment yet, nor is it initialized
    * INSTALLED: The instance is installed into its environment
    * STARTING: The coprocessor is about to be started, its start() method is about to be invoked
    * ACTIVE: Once the start() call returns, the state is set to active
    * STOPPING: The state set just before the stop() method is called
    * STOPPED: Once stop() returns control to the framework, the state of the coprocessor is set to stopped

The _CoprocessorHost_ class maintains all the coprocessor instances and their dedicated environments. The trinity of
_Coprocessor_, _CoprocessorEnvironment_, and _CoprocessorHost_ forms the basis for the classes that implement the advanced
functionality of HBase, provide the life-cycle support for the coprocessors, manage their state, and offer the environment
for them to execute as expected.

#### Coprocessor Loading

You can either configure coprocessors to be loaded in a static way (uses the configuration files and table schemas), or load
them dynamically while the cluster is running (uses only the table schemas). There is also a cluster-wide switch that allows
you to disable all coprocessor loading, controlled by the following two configuration properties:

    * hbase.coprocessor.enabled: Defaults to true, setting this property to false stops the servers from loading any of them 
    * hbase.coprocessor.user.enabled: Defaults to true, controls the loading of user table coprocessors only

##### Loading from Configuration

You can configure globally in _hbase-site.xml_ file, which coprocessors are loaded when HBase starts:

```xml

<property>
    <name>PROCESSOR_TYPE</name>
    <value>SOME_PROCESSOR_FULLY_QUALIFIED_CLASS_NAME</value>
</property>
```

The order of the classes in each configuration property is important, as it defines the execution order. All of these
coprocessors are loaded with the system priority. The possible values for the PROCESSOR_TYPE above are:

| Property | Coprocessor Host | Server Type |
| -------- | ---------------- | ----------- |
| hbase.coprocessor.master.classes| MasterCoprocessorHost | Master Server |
| hbase.coprocessor.regionserver.classes| RegionServerCoprocessorHost | Region Server |
| hbase.coprocessor.region.classes| RegionCoprocessorHost | Region Server |
| hbase.coprocessor.user.region.classes| RegionCoprocessorHost | Region Server |
| hbase.coprocessor.wal.classes| WALCoprocessorHost | Region Server |

When a coprocessor loaded from the configuration fails to start, it will cause the entire server process to be aborted.

##### Loading from Table Descriptor

The other option to define which coprocessors to load is the table descriptor. As this is per table, the coprocessors defined
here are only loaded for regions of that table and only by the region servers hosting these regions. You need to add their
definition to the table descriptor using either:

    * HTableDescriptor.setValue(): You need to create a key that must start with _COPROCESSOR_, and the value has to conform 
      to the following format: 
      `[<path-to-jar>(optional)]|<classname>(required)|[<priority>(optional)][|key1=value1, key2=value2,.. .(optional)]`. 
      The key is a combination of the prefix _COPROCESSOR_, a dollar sign as a divider, and an ordering number, i.e: 
      `COPROCESSOR$1`
    * HTableDescriptor.addCoprocessor(): This will look for the highest assigned number and use the next free one after that

An example of how to add a coprocessor programatically is shown below:

```
HTableDescriptor htd=new HTableDescriptor(TableName.valueOf("testtable"));
        htd.addFamily(new HColumnDescriptor("colfam1"));
        htd.setValue("COPROCESSOR$1","|"+RegionObserverExample.class.getCanonicalName()+"|"+Coprocessor.PRIORITY_USER);

Admin admin=connection.getAdmin();
admin.createTable(htd);
```

Using the `addCoprocessor()` method:

```
HTableDescriptor htd=new HTableDescriptor(tableName)
        .addFamily(new HColumnDescriptor("colfam1"))
        .addCoprocessor(RegionObserverExample.class.getCanonicalName(),null,Coprocessor.PRIORITY_USER,null);

Admin admin=connection.getAdmin();
admin.createTable(htd);
```

Once the table is enabled and the regions are opened, the framework will first load the configuration coprocessors and then
the ones defined in the table descriptor. For table coprocessors there is a configuration property named
_hbase.coprocessor.abortonerror_, indicating what you want to happen if an er‐ ror occurs during the initialization of a
coprocessor class (defaults true).

##### Loading from HBase Shell

If you want to load coprocessors while HBase is running, you can use the table descriptor and the alter call, provided by the
administrative API:
`hbase> alter 'test:usertable', 'coprocessor' => 'file:///test.jar | coprocessor.SequentialIdGeneratorObserver|'`, and find
the result with `hbase> describe 'testqauat:usertable'`. To remove a coprocessor dinamically, you can use
`hbase> alter 'test:usertable', METHOD => 'table_att_unset', NAME => 'coprocessor$1'`.

#### Endpoints

Endpoints solve a problem with moving data for analytical queries, that would benefit from precalculating intermediate
results where the data resides, and just ship the results back to the client.

##### The Service Interface

Endpoints are implemented as an extension to the RPC protocol between the client and server. The payload is serialized as a
Protobuf message and sent from client to server (and back again) using the provided coprocessor services API. In order to
provide an endpoint to clients, a coprocessor generates a Protobuf implementation that extends the Service class. This
service can define any methods that the coprocessor wishes to expose. Using the generated classes, you can communicate with
the coprocessor instances via the following calls, provided by _Table_:

```
CoprocessorRpcChannel coprocessorService(byte[] row)
<T extends Service, R> Map<byte[],R> coprocessorService(
    final Class<T> service,
    byte[] startKey, 
    byte[] endKey, 
    final Batch.Call<T,R> callable) throws ServiceException, Throwable
<T extends Service, R> void coprocessorService(
    final Class<T> service,
    byte[] startKey, 
    byte[] endKey, 
    final Batch.Call<T,R> callable,
    final Batch.Callback<R> callback) throws ServiceException, Throwable
<R extends Message> Map<byte[], R> batchCoprocessorService(
    Descriptors.MethodDescriptor methodDescriptor, 
    Message request,
    byte[] startKey, 
    byte[] endKey, 
    R responsePrototype) throws ServiceException, Throwable
<R extends Message> void batchCoprocessorService(
    Descriptors.MethodDescriptor methodDescriptor,
    Message request, 
    byte[] startKey, 
    byte[] endKey, 
    R responsePrototype,
    Batch.Callback<R> callback) throws ServiceException, Throwable
```

Since Service instances are associated with individual regions within a table, the client RPC calls must ultimately identify
which regions should be used in the service’s method invocations. The coprocessor RPC calls use row keys to identify which
regions should be used for the method invocations. The call options are:

    * Single Region:This is done by calling coprocessorService() with a single row key. This returns an instance of the 
      CoprocessorRpcChannel class, which directly extends Protobuf classes. It can be used to invoke any endpoint call 
      linked to the region containing the specified row. Note that the row does not need to exist: the region selected is 
      the one that does or would contain the given key
    * Ranges of Regions:You can call coprocessorService() with a start row key and an end row key. All regions in the table 
      from the one containing the start row key to the one containing the end row key (inclusive) will be used as the 
      endpoint targets. This is done in parallel up to the amount of threads configured in the executor pool instance in use
    * Batched Regions:If you call batchCoprocessorService() instead, you still parallelize the execution across all regions
      but calls to the same region server are sent together in a single invocation. This will cut down the number of network 
      roundtrips, and is especially useful when the expected results of each endpoint invocation is very small

There is two ways we can invoke a _Service_ from the _Table_ class, clients implement _Batch.Call_ to call methods of the
actual _Service_ implementation instance, the interface’s `call()` method will be called once per selected region, pass‐ ing
the _Service_ implementation instance for the region as a parameter. Clients can optionally implement _Batch.Callback_
to be notified of the results from each region invocation as they complete. The instance’s
`void update(byte[] region, byte[] row, R result)` method will be called with the value returned by `R call(T instance)`
from each region.

##### Implementing Endpoints

Two steps are required to implement an _Endpoint_:

    1. Define the Protobuf service and generate classes (using the protobuf compiler `protoc`)
    2. Extend the generated, custom Service subclass. All the declared RPC methods need to be implemented with the user code.
       This is done by extending the generated class, plus merging in the Coprocessor and CoprocessorService interfaces. 

So if in your protobuffer file you have defined something like this (Step 1):

```protobuf
option java_package = "coprocessor.generated";
option java_outer_classname = "RowCounterProtos";
option java_generic_services = true;
option java_generate_equals_and_hash = true;
option optimize_for = SPEED;
message CountRequest {
}
message CountResponse {
  required int64 count = 1 [default = 0];
}
service RowCountService {
  rpc getRowCount(CountRequest)
      returns (CountResponse);
  rpc getCellCount(CountRequest)
      returns (CountResponse);
}
```

Your code implementing the above should look like this:

```
public class RowCountEndpoint extends RowCounterProtos.RowCountService implements Coprocessor, CoprocessorService {
    @Override
    public void getRowCount(
        RpcController controller,
        RowCounterProtos.CountRequest request, 
        RpcCallback<RowCounterProtos.CountResponse> done) {
        
        RowCounterProtos.CountResponse response = null;
        //TODO implement logic to get count as a long
        done.run(RowCounterProtos.CountResponse.newBuilder().setCount(count).build());
    }
    
    @Override
    public void getCellCount(
        RpcController controller,
        RowCounterProtos.CountRequest request,
        RpcCallback<RowCounterProtos.CountResponse> done) {
        RowCounterProtos.CountResponse response = null;
        //TODO implement logic to get count as a long
        done.run(RowCounterProtos.CountResponse.newBuilder().setCount(count).build());
    }
}
```

After your implementation is complete, you need to add the following to your _hbase-site.xml_ file:

```xml

<property>
    <name>hbase.coprocessor.user.region.classes</name>
    <value>coprocessor.RowCountEndpoint</value>
</property>
```

To make use of the above functionality, you need to use the  _Table_ interface:

```
Table table = connection.getTable(tableName);
try {
  final RowCounterProtos.CountRequest request = RowCounterProtos.CountRequest.getDefaultInstance();
  Map<byte[], Long> results = table.coprocessorService(RowCounterProtos.RowCountService.class, null /*start*/, null /*end*/,
    new Batch.Call<RowCounterProtos.RowCountService, Long>() {
        public Long call(RowCounterProtos.RowCountService counter) throws IOException {
            BlockingRpcCallback<RowCounterProtos.CountResponse> rpcCallback =
                new BlockingRpcCallback<RowCounterProtos.CountResponse>();
            counter.getRowCount(null, request, rpcCallback); 
            RowCounterProtos.CountResponse response = rpcCallback.get();
            return response.hasCount() ? response.getCount() : 0;
          }
    });
    // TODO handle the response
} catch (Throwable throwable) {
  throwable.printStackTrace();
}
```

Using batch calls instead:

```
final CountRequest request = CountRequest.getDefaultInstance(); 
Map<byte[], CountResponse> results = table.batchCoprocessorService(
    RowCountService.getDescriptor().findMethodByName("getRowCount"),
    request, 
    HConstants.EMPTY_START_ROW, 
    HConstants.EMPTY_END_ROW,
    CountResponse.getDefaultInstance());
```

Use the later form when you have many regions per server, and the returned data is very small, then the cost of the RPC
roundtrips are noticeable. To use the _CoprocessorRpcChannel_ call:

```
CoprocessorRpcChannel channel = table.coprocessorService(Bytes.toBytes("row1"));
RowCountService.BlockingInterface service = RowCountService.newBlockingStub(channel);
CountRequest request = CountRequest.newBuilder().build();
CountResponse response = service.getCellCount(null, request);
```

With the proxy reference, you can invoke any remote function defined in your derived _Service_ implementation from within
client code, and it returns the result for the region that served the request.

#### Observers

Observers are like triggers in RDBMS, they differ to endpoints in that they are not only running in the context of a region.
They can run in different parts of the system and react to events triggered by clients, but also implicitly by servers.
Observers use pre-defined hooks into the server processes, so you can't add your own ones. They also act on the server side
only, with no connection to the client. What you can do though is combine an endpoint with an observer for region-related
functionality, exposing observer state through a custom RPC API.

##### The ObserverContext Class

For the callbacks provided by the _Observer_ classes, there is a special context handed in as the first parameter to all
calls: an instance of the _ObserverContext_ class. It provides access to the current environment, and the ability to indicate
to the coprocessor framework what it should do after a callback is completed. Methods provided by the context class:

    * E getEnvironment(): Returns the reference to the current coprocessor environment 
    * void prepare(E env): Prepares the context with the specified environment 
    * void bypass(): When your code invokes this method, the framework returns your value instead of the calling method value
    * void complete(): To indicate that any further processing and coprocessors in the execution chain, can be skipped
    * boolean shouldBypass(): Used internally by the framework to check on the bypass flag 
    * boolean shouldComplete(): Used internally by the framework to check on the complete flag
    * static <T extends CoprocessorEnvironment> ObserverContext<T> createAndPrepare(T env, ObserverContext<T> ctx): Static 
      function to initialize a context, it will create a new instance when the provided context is null

The `complete()` call influences the execution chain of the coprocessors, while the `bypass()` call stops any further default
processing on the server within the current observer.

##### The RegionObserver Class

Operations in this class can be divided into two groups: region life-cycle changes and client API calls, there is a generic
callback for many operations of both kinds:

```
enum Operation {
ANY, GET, PUT, DELETE, SCAN, APPEND, INCREMENT, SPLIT_REGION, MERGE_REGION, BATCH_MUTATE, REPLAY_BATCH_MUTATE, COMPACT_REGION
}

postStartRegionOperation(ObserverContext<RegionCoprocessorEnvironment> ctx, Operation operation)
postCloseRegionOperation(ObserverContext<RegionCoprocessorEnvironment> ctx, Operation operation)
```

These methods in a RegionObserver are invoked when any of the possible Operations listed is called.

###### Handling Region Life-Cycle Events

The Lifecycle state changes of a region is as follows `unassigned --> pending open --> open --> pending close --> closed`
and coprocessors can hook in the pending open, open and pending close state.

*State: pending open*

A region is in this state when it is about to be opened. The following callbacks in order of their invocation are available:

```
postLogReplay(...) // triggered dependent on what WAL recovery mode is configured: distributed log splitting or log replay
preOpen(...) // just before the region is opened
preStoreFileReaderOpen(...) //before the store files are opened in due course
postStoreFileReaderOpen(...) // after the store files are opened in due course
preWALRestore(...) / postWALRestore(...) // the WAL being replayed
postOpen(...) // just after the region was opened
```

*State: open*
A region is considered open when it is deployed to a region server and fully operational. The possible hooks are:

```
preFlushScannerOpen(...)
preFlush(...) / postFlush(...)
preCompactSelection(...) / postCompactSelection(...)
preCompactScannerOpen(...)
preCompact(...) / postCompact(...)
preSplit(...)
preSplitBeforePONR(...)
preSplitAfterPONR(...)
postSplit(...)
postCompleteSplit(...) / preRollBackSplit(...) / postRollBackSplit(...)
```

Obviously, the pre calls are executed before, while the post calls are executed after the respective operation. The hooks for
flush, compact, and split are directly linked to the matching region housekeeping functions.

*State: pending close*

Occurs when the region transitions from open to closed. Just before and after the region is closed these hooks are executed:

```
preClose(...,  boolean abortRequested)
postClose(..., boolean abortRequested)
```

###### Handling Client API Events

All client API calls are explicitly sent from a client application to the region server and there are hooks for these
operations too. For example, the `Table.put()` operation has a `prePut(...)` and a `postPut(...)` hooks which has their order
of execution, check the documentation for more details. Example of usage:

```
public class ObserverStatisticsCustom  implements RegionObserver{
    @Override
    public void preOpen(ObserverContext<RegionCoprocessorEnvironment> observerContext) throws IOException {
        someMethod();
    }
    @Override
    public void postOpen(ObserverContext<RegionCoprocessorEnvironment> observerContext) {
      someOtherMethod();
    }
}
```

###### The RegionCoprocessorEnvironment Class

The environment instances provided to a coprocessor that is implementing the _RegionObserver_ interface are based on the
_RegionCoprocessorEnvironment_ class, which in turn is implementing the _CoprocessorEnvironment_ interface. Specific methods
provided by the _RegionCoprocessorEnvironment_ class:

    * getRegion(): Returns a reference to the region the current observer is associated with
    * getRegionInfo(): Get information about the region associated with the current coprocessor instance
    * getRegionServerServices(): Provides access to the shared RegionServerServices instance
    * getSharedData(): All the shared data between the instances of this coprocessor

The `getRegionInfo()` method returns a _HRegionInfo_ instance, which has some very useful methods:

```
byte[] getStartKey()
byte[] getEndKey()
byte[] getRegionName()
boolean isSystemTable()
int getReplicaId()
```

The `RegionServerServices()` call returns a _RegionServerServices_ class, which also has a lot of methods (check javadocs).

###### The BaseRegionObserver Class

This class can be used as the basis for all your observer-type coprocessors. It has placeholders for all methods required by
the _RegionObserver_ interface. They are all left blank, so by default nothing is done when extending this class.

##### The MasterObserver Class

This subclass of Coprocessor handles all possible callbacks the master server may initiate. The MasterObserver class provides
hooks on DDL alike operations such as `pre(post)CreateTable`, `pre(post)AddColumn` and more. Check the java docs for more
details. The methods are grouped roughly into table and region, namespace, snapshot, and server related calls.

###### The MasterCoprocessorEnvironment Class

The MasterCoprocessorEnviron meant is wrapping MasterObserver instances. It also implements the CoprocessorEnvironment
interface, giving you access to the `getTable()` call. Specific method provided by the _MasterCoprocessorEnvironment_ class:

    * getMasterServices(): Provides access to the shared MasterServices instance

The above method provides access the shared master services instance, which exposes many functions of the Master admin API
such as table CRUD operations, namespace related methods. Check the java docs for more details.

###### The BaseMasterObserver Class

Either you can implement a MasterObserver on the interface directly, or you can extend the BaseMasterObserver class instead,
which implements the interface while leaving all callback functions empty. You need to add the following to the
_hbase-site.xml_ file for the coprocessor to be loaded by the master process (a restart is required):

```xml

<property>
    <name>hbase.coprocessor.master.classes</name>
    <value>coprocessor.MasterObserverExample</value>
</property>
```

The environment is wrapped in an _ObserverContext_, you have the same extra flow controls, exposed by the `bypass()` and
`complete()` methods. You can use them to explicitly disable certain operations or skip subsequent coprocessor execution.

###### The BaseMasterAndRegionObserver Class

The _BaseMasterAndRegionObserver_ class is a combination of the _BaseRegionObserver_ and the _MasterObserver_. This class is
only useful to run on the HBase Master.

##### The RegionServerObserver Class

You can run code within the region servers using the _RegionServerObserver_ class. It exposes hooks spanning many regions and
tables. Methods of interest:

    * postCreateReplicationEndPoint(...): Invoked after the server has created a replication endpoint
    * preMerge(...), postMerge(...): Called when two regions are merged.
    * preMergeCommit(...), postMergeCommit(...): Called after preMerge() and before postMerge().
    * preRollBackMerge(...), postRollBackMerge(...): Called when a region merge fails and merge transaction is rolled back
    * preReplicateLogEntries(...), postReplicateLogEntries(...): To allow special treatment of each log entry
    * preRollWALWriterRequest(...), postRollWALWriterRe quest(...): Wrap the rolling of WAL files
    * preStopRegionServer(...): Called when the Stoppable method stop() is called

###### The RegionServerCoprocessorEnvironment Class

This class wraps the _RegionServerObserver_ instances. Implements the _CoprocessorEnvironment_ interface. Specific method
provided by the _RegionServerCoprocessorEnvironment_ class

    * getRegionServerServices(): Provides access to the shared RegionServerServices instance

###### The BaseRegionServerObserver Class

This is an empty implementation of the RegionSer verObserver interface, so you can overwrite the necessary methods only.

##### The WALObserver Class

The write-ahead log or WAL, offers a manageable list of callbacks, namely the following two:

    * preWALWrite(...), postWALWrite(...): Wrap the writing of log entries to the WAL allowing access to the full edit record

Since you receive the entire record in these methods, you can influence what is written to the log.

###### The WALCoprocessorEnvironment Class

There is a specialized environment that is provided as part of the callbacks. Here it is an instance of the
_WALCoprocessorEnvironment_ class. It also extends the _CoprocessorEnvironment_ interface, giving you access to the
`getTable()` call to access data from within your own implementation. Specific method provided by the
_WALCoprocessorEnvironment_ class:

    * getWAL(): Provides access to the shared WAL instance

The methods available from the WAL interface are:

```
void registerWALActionsListener(final WALActionsListener listener) 
boolean unregisterWALActionsListener(final WALActionsListener listener)
byte[][] rollWriter() throws FailedLogCloseException, IOException 
byte[][] rollWriter(boolean force) throws FailedLogCloseException, IOException
void shutdown() throws IOException
void close() throws IOException
long append(HTableDescriptor htd, 
    HRegionInfo info, 
    WALKey key, 
    WALEdit edits,
    AtomicLong sequenceId, 
    boolean inMemstore, 
    List<Cell> memstoreKVs) throws IOException
void sync() throws IOException
void sync(long txid) throws IOException
boolean startCacheFlush(final byte[] encodedRegionName)
void completeCacheFlush(final byte[] encodedRegionName)
void abortCacheFlush(byte[] encodedRegionName)
WALCoprocessorHost getCoprocessorHost()
long getEarliestMemstoreSeqNum(byte[] encodedRegionName)
```

###### The BaseWALObserver Class

The _BaseWALObserver_ class implements the _WALObserver_ interface, use this class to implement your own or the interface
directly.

##### The BulkLoadObserver Class

Class used during bulk loading operations, as triggered by the HBase supplied _completebulkload_ tool, contained in the
server JAR file. During that operation the available callbacks are triggered:

    * prePrepareBulkLoad(...): Invoked before the bulk load operation takes place
    * preCleanupBulkLoad(...): Called when the bulk load is complete and clean up tasks are performed

##### The EndPointObserver Class

This observer shares the _RegionCoprocessorEnvironment_, because endpoints runs in a region context. Callback methods are:

    * preEndpointInvocation(...), postEndpointInvocation(...): callbacks to wraps the server side execution whenever an 
      endpoint method is called from a client

The client can replace the given _Message_ instance to modify the outcome of the endpoint method. The server-side call is
aborted completely in case of failure.