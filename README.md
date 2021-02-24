# HBase The definitive guide

## Table of contents:

1. [Chapter 1: Introduction](#Chapter1)
2. [Chapter 1: Installation](#Chapter2)
3. [Chapter 3: Client API: The Basics](#Chapter3)

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
OperationWithAttributes class is ex‐ tending the above Operation class, implements the Attributes inter‐ face, and is adding
the following methods, which are used in conjunction:

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
in the exact same order. The _isStale()_ method is used to check if we have retrieved data from a replica, not the
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

the method that is used to retrieve specific values from a HBase table: `Result get(Get get) throws IOException`. A _get()_
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
_getMaxResultsPerColumnFamily()_, _setMaxResultsPerColumnFamily()_, _getRowOffsetPerColumnFamily()_, and
_setRowOffsetPerColumnFamily()_ works in tandem to allow the client to page through a wide row, setting the maximum amount of
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

The result class provides you with the means to ac‐ cess everything that was returned from the server for the given row and
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
for example _getColumnCells()_ method you ask for multiple values of a specific column, allowing you to get multiple versions
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

Another similarity to the _put()_ calls is that you can ask for more than one row using a single request with:
`Result[] get(List<Get> gets) throws IOException`

This is used as follows:

```
List<Get> gets = new ArrayList<Get>();
// Some code to add gets to the list
Result[] results = table.get(gets);
// More code
```

Errors are reported back to you differently than with the Puts. The _get()_ call either returns the said array, matching 
the same size as the given list by the gets parameter, or throws an exception and not returning a result at all.

#### Delete Method