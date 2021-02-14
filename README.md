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

The HBase Shell is (J)Ruby’s IRB with some HBase-related commands added. Find more by typing _help_ to see a listing of 
shell commands and options. 


## Chapter 3: Client API: The Basics<a name="Chapter3"></a>
