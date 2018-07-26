# Hadoop Application Architectures

## Table of Contents

1. [Chapter 1: Data Modeling in Hadoop](#Chapter1)
2. [Chapter 2: Data Movement](#Chapter2)
3. [Chapter 3: Processing Data in Hadoop](#Chapter3)
4. [Chapter 4: Common Hadoop Processing Patterns](#Chapter4)
5. [Chapter 5: Graph Processing on Hadoop](#Chapter5)
6. [Chapter 6: Orchestration](#Chapter6)
7. [Chapter 7: Near-Real-Time Processing with Hadoop](#Chapter7)
8. [Chapter 8: Clickstream Analysis](#Chapter8)
9. [Chapter 9: Fraud Detection](#Chapter9)
10. [Chapter 10: Data Warehouse](#Chapter10)

## Chapter 1: Data Modeling in Hadoop<a name="Chapter1"></a>
There are many factors that you should take into consideration before dumping your data into Hadoop:

    * Data storage formats: Several formats are supported (HBase, Hive...)
    * Multitenancy: Clusters can support multiple users and groups
    * Schema design: Directory structures and output of data process (HBase, Hive...) can be a decission factor
    * Metadata management: Metadata related to the stored data is often as important as the data itself
    
### Data Storage Options
There is no such thing as a standard data storage format in Hadoop, although Hadoop provides built-in support for a 
number of formats optimized for Hadoop storage and processing, including data generated during data processing and 
derived data from processing. Major considerations:

    * File Format: The different formats have different strengths that make them more or less suitable depending on 
    the application and source-data types
    * Compression: Compression rate, speed of uncrompression and the ability to split the compressed files are 
    important considerations to make
    * Data storage system: The underlying storage manager like Impala, Hive, HDFS or HBase is also decisive
    
#### Standard File Formats
In general, it’s preferable to use one of the Hadoop-specific container formats for storing data in Hadoop, 
considerations for storing standard file formats in Hadoop:

    * Text data: Consider the organization of the files in the filesystem and compression format for the files. Keep 
    in mind that there is an overhead of type conversion associated with storing data in text format (from string to 
    int for example). Select high compression for archival purposes and a splittable format for parallel processing 
    jobs. SequenceFiles or Avro are the preferred format for most file types, including text as they provide 
    functionality to support splittable compression
    * Structured text data: XML and JSON are tricky because they are not easily splittable and Hadoop does not 
    provide a built-in InputFormat for either. The recomendation is to use a container format such as Avro and use 
    specific libraries to process the files
    * Binary data: Using a container format such as SequenceFile is preferred. If the splittable unit of binary data 
    is larger than 64 MB, you may consider putting the data in its own file without a container format
    
#### Hadoop File Types

    * Splittable compression: Allows large files to be split for input to MapReduce and other types of jobs
    * Agnostic compression: The file can be compressed with any compression codec, without readers having to know the
     codec, possible because the codec is stored in the header metadata of the file format
     
##### File-based data structures
SequenceFile is the format most commonly employed in implementing Hadoop jobs. They store data as binary key-value 
pairs with three formats available for records stored within SequenceFiles:

    * Uncompressed: Less efficient for input/output (I/O) and take up more space on disk than the same compressed form
    * Record-compressed: Each record is compressed on the file
    * Block-compressed: Block compression provides better compression ratios compared to record-compressed 
    SequenceFiles, and is generally the preferred compression option for SequenceFiles (unrelated to the HDFS or 
    filesystem block)
    
Every SequenceFile uses a common header format containing basic metadata about the file, such as the compression 
codec used, key and value class names, user-defined metadata, and a randomly generated sync marker per block of data 
(which allow for seeking to random points in the file, and is key to facilitating splittability). A common use case for 
SequenceFiles is as a container for smaller files.

##### Serialization Formats
Serialization allows data to be converted into a format that can be efficiently stored as well as transferred across 
a network connection and its associated to interprocess communication (remote procedure calls, or RPC) and data storage.
The main serialization format utilized by Hadoop is Writables, but other serialization frameworks seeing increased 
use within the Hadoop eco‐system, including Thrift, Protocol Buffers, and Avro (best suited as it was specifically 
created to address limitations of Hadoop Writables).

    * Thrift: Framework for implementing cross-language interfaces to services, it does not support internal 
    compression of records, it’s not splittable and it lacks native MapReduce support
    * Protocol Buffers: Similar to Thrift (cross-language, doesn't support record compression, not splittable, no 
    native support)
    * Avro: Portable language-neutral data serialization system, described through a language-independent schema. 
    Avro stores the schema in the header of each file, it’s self-describing and supports schema evolution (the schema
     used to read a file does not need to match the schema used to write the file). Avro also has a sync marker to 
     separate blocks in the file a
     
##### Columnar Formats
Benefits of columnar formats over earlier row-oriented systems:

    * Skips I/O and decompression (if applicable) on columns that are not a part of the query
    * Works well for queries that only access a small subset of columns
    * Generally very efficient in terms of compression due to data being more similar within the same column, than it
     is in a block of rows
    * Suited for data-warehousing-type applications where users want to aggregate certain columns over 
    a large collection of records
    
Formats:

    * RCFile: Used as a Hive storage format. The RCFile format was developed to provide fast data loading, fast query 
    processing, and highly efficient storage space utilization.
    * ORC: Solves the shortcomings of the RCFile format around query performance and storage efficiency. The ORC format:
        - Provides lightweight, always-on compression provided by type-specific readers and writers
        - Allows predicates to be pushed down to the storage layer so that only required data is brought back in queries
        - Supports the Hive type model, including new primitives such as decimal and complex types
        - Splittable
    * Parquet: Similar to ORC, provides efficient compression, support nested data structures, stores full metadata 
    at the end of the file (self-documented), compatible with Avro and Thrift APIs, uses  efficient and extensible
    encoding schemas

Columnar formats do not work well in the event of failure, leading to incomplete rows. Sequence files will be readable 
 to the first failed row, but will not be recoverable after that row. Avro, in the event of a bad record, the read will 
 continue at the next sync point, so failures only affect a portion of a file.

##### Compression
Compression adds CPU load, for most cases this is more than offset by the savings in I/O. Not all compression formats
 are splittable:
 
    * Snappy: Not splittable, high compression speeds with reasonable compression
    * LZO: Optimized for speed as opposed to size, is plittable (requires an indexing step). Good choice for things 
    like plain-text files that are not being stored as part of a container format. Requires a separate install
    * Gzip: Good compression, low write performance, read performance similar to snappy and not splittable
    * bzip2: Excellent compression, slower than snappy, splittable. Has a big read/write performance cost. Not an 
    ideal codec for Hadoop storage, unless your primary need is reducing the storage foot‐print
    
In general, any compression format can be made splittable when used with container file formats (Avro, SequenceFiles,
 etc.) that compress blocks of records or each record individually. Recommendations:

    * Enable compression of MapReduce intermediate output
    * Pay attention to how data is ordered. Often, ordering data so that like data is close together will provide 
    better compression levels
    * Consider using a compact file format with support for splittable compression, such as Avro
    
#### HDFS Schema Design
Hadoop’s Schema-on-Read model does not impose any requirements when loading data into Hadoop. Although some order is 
still desirable due to:
    
    * A standard directory structure makes it easier to share data between teams working with the same data sets
    * Enforcing access and quota controls to prevent accidental deletion or corruption
    * Conventions regarding staging data will help ensure that partially loaded data will not get accidentally 
    processed as if it were complete
    * Standardized organization of data allows for reuse of some code that may process the data
    * Some tools in the Hadoop ecosystem sometimes make assumptions regarding the placement of data. It is often 
    simpler to match those assumptions when you are initially loading data into Hadoop
    
Keep usage patterns in mind when designing a schema. Different data processing and querying patterns work better with
 different schema designs.
 
##### Location of HDFS Files
Standard locations make it easier to find and share data between teams. Standard locations:

    * /user/<username>: Data, JARs, and configuration files that belong only to a specific user
    * /etl: Data in various stages of being processed by an ETL. This directory will be readable and writable by ETL 
    processes (they typically run under their own user) and members of the ETL team. An example structure will look 
    similar to /etl/<group>/<application>/<process>/{input,processing,output,bad}
    * /tmp: Temporary data generated by tools or shared between users,typically readable and writable by everyone
    * /data: Data sets that have been processed and are shared across the organization. Strict read/write policies 
    are usually enforced here (usually only automated ETL processes are typically allowed to write them)
    * /app: Includes everything required for Hadoop applications to run, except data, like JAR files, 
    Oozie workflow definitions, Hive HQL files, and more. This directory should have a subdirectory for each group 
    and application, similar to the structure used in /etl. This will look similar to: 
    /app/<group>/<application>/<version>/<artifact directory>/<artifact>
    * /metadata: Stores metadata. Ttypically readable by ETL jobs but writable by the user used for ingesting data 
    into Hadoop
    
##### Advanced HDFS Schema Design
There are a few strategies to best organize your data:

    * Partitioning: Unlike traditional data warehouses, HDFS doesn’t store indexes on the data which means that every 
    query will have to read the entire data set. Breaking up the data set into smaller subsets, or partitions allow 
    queries to read only the specific partitions they require, reducing the amount of I/O and improving query times 
    significantly. When placing the data in the filesystem, you should use the following directory format: 
    <data set name>/<partition_column_name=partition_column_value>/{files}
    This directory structure is understood by various tools (HCatalog, Hive, Impala, and Pig)
    * Bucketing: Similar to the hash partitions used in many relational databases. A hash (or similar) function is 
    applied to a column value (i.e. a postcode) to find the bucket it should be placed on, a good average bucket size
    is a few multiples of the HDFS block size. On SQL alike joins, when both the data are bucketed on the join key 
    and the number of buckets of one data set is a multiple of the other, it is enough to join corresponding buckets
    individually without having to join the entire data sets. Sorted data provides even greater gains. It is 
    recommended to use both sorting and bucketing on all large tables that are frequently joined together, using the 
    join key for bucketing
    * Denormalizing:  In cases when there are multiple common query patterns and it is challenging to decide on one 
    partitioning key, you have the option of storing the same data set multiple times, each with different physical 
    organization. While bucketing and sorting do help there, another solution is to create data sets that are 
    prejoined (preaggregated)
    
### HBase Schema Design
HBase can be seeing as a huge hash table. Hash tables supports put, get, delete, iterations, value increment... 

#### Row key
Considerations to correctly design a row key:

    * Record retrieval: HBase records can have an unlimited number of columns, but only a single row key. Keep in 
    mind that a get operation of a single record is the fastest operation in HBase, thus designing the HBase schema in 
    such a way that most common uses of the data will result in a single get operation will improve performance 
    (denormalized data is helpful here)
    * Distribution: The row key determines how records for a given table are scattered throughout various regions 
    of the HBase cluster. Rows are sorted in HBase, regions stores a range of these sorted row keys and each region is 
    pinned to a region server
    * Block cache:The block cache is a least recently used (LRU) cache that caches data blocks in memory. HBase reads
    records in chunks of 64 KB from the disk called HBase block. When an HBase block is read from the disk, it will 
    be put into the block cache (which can be bypassed if you desire). The idea behind the caching is that recently 
    fetched records (and those in the same HBase block as them) have a high likelihood of being requested again in 
    the near future. A poor choice of row key can lead to suboptimal population of the block cache
    * Ability to scan: A wise selection of row key can be used to co-locate related records in the same region, which
    is very beneficial in range scans since HBase will have to scan a limited number of regions to obtain the results
    * Size: The size of your row key will determine the performance of your workload, the longer the row key, the 
    more I/O the compression codec has to do in order to store it, same for column names.
    * Readability: Start with something human-readable for your row keys, even more so if you are new to HBase
    * Uniqueness: A row key is equivalent to a key in our hash table analogy
    
#### Timestamp
In HBase, timestamps serve a few important purposes:
  
    * Determines which records are newer in case of a put request to modify the record
    * Determines the order in which records are returned when multiple versions of a single record are requested
    * Used to decide if a major compaction is going to remove the out-of-date record in question because the 
    time-to-live (TTL) when compared to the timestamp has elapsed. “Out-of-date” means that the record value has 
    either been overwritten by another put or deleted

#### Hops
Hops are the number of synchronized get requests required to retrieve the requested information from HBase. In general, 
although it’s possible to have multihop requests with HBase, it’s best to avoid them through better schema design.

#### Tables and Regions
The number of tables and number of regions per table in HBase also impacts performance and distribution of data:

    * Put performance: All regions in a region server receiving put requests will have to share the region server’s 
    memstore. A memstore is a cache structure present on every HBase region server that caches the writes being sent 
    to that region server and sorts them in before it flushes them when certain memory thresholds are reached. 
    The more regions that exist in a region server, the less memstore space is available per region. This may result 
    in smaller flushes, which in turn may lead to smaller and more HFiles and more minor compactions. The default 
    configuration will set the ideal flush size to 100 MB; the size of your memstore divided by 100 MB should be the 
    maximum number of regions you can reasonably put on that region server. 
    * Compaction time: A larger region takes longer to compact. The empirical upper limit of a region is around 20 GB. 
    It is recommended to preselect the region count of the table to avoid random region splitting and suboptimal 
    region split ranges. 

#### Using Columns
HBase stores data in a format called HFile. Each column value gets its own row in HFile, each column value gets its 
own row in HFile. The amount of space consumed on disk plays a nontrivial role in your decision on how to structure 
your HBase schema, in particular the number of columns. 

#### Using Column Families
A column family is a container for columns, each column family has its own set of HFiles and gets compacted 
independently of other column families in the same table.  The main reason to use more than one column family is when 
the operations being done and/or the rate of change on a subset of the columns of a table is significantly different
from the other columns (optimizes compaction cost and block cache usage).
 
#### Time-to-Live
Built-in feature of HBase that ages out data based on its timestamp, if on a major compaction the timestamp is older 
than the specified TTL in the past, the record in question is removed.

### Managing Metadata
Metadata, in general, refers to data about the data, it can refer to:

    * Logical data sets information, which is usually stored in a separate metadata repository
    * Information about files on HDFS, which is usually stored and managed by Hadoop NameNode
    * HBase metadata, which is stored and managed by HBase itself.
    * Metadata about data ingest and transformations
    * Data set statistics, useful for optimizing execution plans and data analysts

Metadata allows you to interact with your data through the higher-level logical abstraction of a table rather than as
a mere collection of files on HDFS or a table in HBase. It also supplies information about your data and allows data 
management tools to “hook” into this metadata for data discovery purposes as well as tracks lineage.

#### Where to store Metadata?
Hive stores this metadata in a relational database called the Hive metastore. Hive also includes a service called the
 Hive metastore service that interfaces with the Hive metastore database. More projects wanted to use the same 
 metadata that was in the Hive metastore, like HCatalog, which allows other tools to integrate with the Hive 
 metastore. Hive metastore can be deployed in three modes: embedded metastore, local metastore, and remote 
 metastore (recommended).
 
#### Other Ways of Storing Metadata
Other ways to store metadata are:

    * Embedding metadata in file paths and names: for example name of the data set, name of the partition column, 
    and the various values of the partitioning column. 
    * Storing the metadata in HDFS: One option to store metadata is to create a hidden directory, inside the directory 
    containing the data in HDFS (i.e. schema of the data in an Avro schema file)
    
## Chapter 2: Data Movement<a name="Chapter2"></a>
Most applications implemented on Hadoop involve ingestion of disparate data types (relational databases and 
mainframes, logs, machine-generated data, event data, files being imported from existing enterprise data storage 
systems) from multiple sources and with differing requirements for frequency of ingestion.

### Data Ingestion Considerations
#### Timeliness of Data Ingestion
Timeliness of data ingestion is the time lag between the data is available for ingestion to when it’s accessible to 
tools in the Hadoop ecosystem. It’s recommended to use one of the following classifications before designing the 
ingestion architecture for an application:

    * Macro batch: 15 minutes to hours, or even a daily job
    * Microbatch: fired off every 2 minutes or so, but no more than 15 minutes in total.
    * Near-Real-Time Decision Support: This is considered to be “immediately actionable” between 2 min and 2 sec
    * Near-Real-Time Event Processing: Under 2 seconds, and can be as fast as a 100-millisecond range 
    * Real Time: Anything under 100 milliseconds

Use hadoop CLI tools or sqoop to ingest data with less strict timeliness requirements, and consider kafka or flume when 
moving towards real time (we need to think about memory first and permanent storage second). Use tools like Storm or 
Spark Streaming stream for processing.

#### Incremental Updates
HDFS has high read and write rates because of its ability to parallelize I/O to multiple drives. The downside to HDFS
is the inability to do appends or random writes to files after they’re created. Consider the following:
 
    * Create periodic process to join multiple small files, using a long consecutive scan to read a single file is 
    faster than performing many seeks to read the same information from multiple files
    * To update files, try to write a “delta” file that includes the changes that need to be made to the 
    existing file and create a compaction job to handle modifications, or consider HBase for this
    
In a compaction job, the data is sorted by a primary key. If the row is found twice, then the data from the newer 
delta file is kept and the data from the older file is not. The results of the compaction process are written to 
disk, and when the process is complete the resulting compaction data will replace the older, uncompacted data.

#### Access Patterns
Recommendation for cases where simplicity, best compression, and highest scan rate are called for, HDFS is the 
default selection. When random access is of primary importance, HBase should be the default, and for search 
processing you should consider Solr.

#### Original Source System and Data Structure

    * Read speed of the devices on source systems: To maximize read speeds, make sure to take advantage of as many 
    disks as possible on the source system (Disk I/O is often a major bottleneck in any processing pipeline).
    * Original file type: Consider Avro instead of CSV, find the most suitable format for the tool that will access 
    that data
    * Compression: Compression comes at the cost of more CPU usage, and possibly unability to split files. Use 
    splittable container formats (copy the compressed file to Hadoop and convert the files in a post-processing step)
    * Relational database management systems: Sqoop (batch tool) is the preferred tool of choice, if timeliness is a 
    concern, consider flume of kafka. In sqoop, every data node connects to the RDBMS so if network is a bottlenech, 
    consider to dump the data in an RDBMS file and ingest if from an edge node
    * Streaming data: Flume or Kafka are highly recommended
    * Logfiles: Ingest logfiles by streaming the logs directly to a tool like Flume or Kafka, instead of reading the 
    logfiles from disk as they are written
    
#### Transformations
Transformation refers to making modifications on incoming data, distributing the data into partitions or buckets, or 
sending the data to more than one store or location. The same advices holds here batch when the timeliness is not 
that much of a concern and kafka, flume, spark for stream. Configure two output directories: one for records that 
were processed successfully and one for failures. Flume has interceptors (Java class that allows for in-flight 
enrichment of event data) and selectors (to send events to different endpoints) to deal with this problem.

#### Network Bottlenecks
If the network is the bottle‐ neck, it will be important to either add more network bandwidth or compress the data 
over the wire. 

#### Network Security
Encrypt data on the wire if it needs to reach outside the company network boundaries (Flume provides native support).

#### Push or Pull

    * Sqoop: Pull solution to transfer data from RDBMS to Hadoop and viceversa
    * Flume: Pushes events through a pipeline (composed of many different types)
    
#### Failure Handling
In distributed computing, the failure scenario has to be consider. For example an `hdfs put ...` failure scenario can
be handled by having multiple local filesystem directories that represent different bucketing in the life cycle of the
file transfer process. Some systems like flume or kafka might produce duplicate records in case of failure so the 
system has to account for this possibility.

### Data Ingestion Options
#### File Transfers
The simplest and fastest way is using the cli tools, and should be considered as the first option when you are designing
 a new data processing pipeline with Hadoop:

    * It is an all or nothing batch processing approach
    * By default single-threaded, not parallelizable
    * From Filesystem to HDFS
    * Applying transformations to data is not supported
    * Byte-to-byte transfer type, any data format supported
    
##### HDFS client commands
When you use the put command there are normally two approaches: the double-hop (copying the file to a hadoop edge 
node filesystem and then to HDFS) and single-hop (which requires that the source device is mountable, for example a 
NAS or SAN, and the put command can read directly from the device and write the file directly to HDFS).

##### Mountable HDFS
There are options to allow clients to interact with HDFS like if it was the normal filesystem, with POSIX support 
although random writes not supported. Example of these options:

    * Fuse-DFS: Involves a number of hops between client applications and HDFS, which can impact performance
    * NFSv3: The design involves an NFS gateway server that streams files to HDFS using the DFSClient (preferred), 
    not suitable for large data volume transfers
    
#### Considerations for File Transfers versus Other Ingest Methods
Some considerations to take into account when you’re trying to determine whether a file transfer is acceptable, or 
whether you should use a tool such as Flume: Do you need to ingest data into multiple locations? Is reliability 
important? Is transformation of the data required before ingestion? If the answer to the questions is yes, Flume or 
kafka are probably a better fit.

#### Sqoop: Batch Transfer Between Hadoop and Relational Databases
Sqoop generates map-only MapReduce jobs where each mapper connects to the database using a Java database connectivity
 (JDBC) driver, selects a portion of the table to be imported, and writes the data to HDFS. 
 
##### Choosing a split-by column
By default, Sqoop will use four mappers and will split work between them by taking the minimum and maximum values of
the primary key column and dividing the range equally among the mappers. The `split-by` parameter lets you specify 
a different column for splitting the table between mappers, and num-mappers lets you control the number of mappers. 
Use this option to avoid Data skew.

##### Using database-specific connectors whenever available
Different RDBMSs support different dialects of SQL language.

##### Using the Goldilocks method of Sqoop performance tuning
In most cases, Hadoop cluster capacity will vastly exceed that of the RDBMS. If Sqoop uses too many mappers, Hadoop 
will effectively run a denial-of-service attack against your database. Start with a very low number of mappers and 
gradually increase it.

##### Loading many tables in parallel with fair scheduler throttling
A common use case is to have to ingest many tables from the same RDBMS. There are two different approaches:

    * Load the tables sequentially: Not optimal, mappers can stay iddle
    * Load the tables in parallel: Uses resources more effectively, but adds complexity

##### Diagnosing bottlenecks
If adding more mappers has little impact on ingest rates, there is a bottleneck somewhere in the pipe, for example:

    * Network bandwidth: If the network limit has been reached, adding more mappers will increase load on the database,
    but will not improve ingest rates
    * RDBMS: Check the query generated by the mappers. Sqoop in incremental mode uses indexes. When Sqoop is used to 
    ingest an entire table, full table scans are typically preferred. If multiple mappers are competing for access to
    the same data blocks, you will also see lower throughput
    * Data skew: Sqoop will look for the highest and lowest values of the PK and divide the range equally between 
    the mappers, use a different column if the data is skewed
    * Connector: Not specific connectors works worst
    * Hadoop: Verify that Sqoop’s mappers are not waiting for task slots, check disk I/O and CPU utilization
    * Inefficient access path: If you specify the split column, it is important that this column is either the 
    partition key or has an index
    
##### Keeping Hadoop updated
If we wish to update data, we need to either replace the data set, add par‐ titions, or create a new data set by 
merging changes. When the table is big and takes a long time to ingest, we prefer to ingest only the modifications 
which requires the ability to identify such modifications. Sqoop supports two methods for this:

    * Sequence ID: Sqoop can track the last ID it wrote to HDFS, and ingest only the newer ones
    * Timestamp: Sqoop can store the last timestamp it wrote to HDFS, and ingest rows with a newer one
    
When running Sqoop with the --incremental flag, you can reuse the same directory name, so the new data will be loaded as
additional files in the same directory although this is not recommended. When the incremental ingest contains updates to
existing rows, we need to merge the new data set with the existing one with the command sqoop-merge.

#### Flume: Event-Based Data Collection and Processing
Flume is a distributed, reliable, and available system for the efficient collection, aggregation, and movement of
streaming data. Main components:

    * Sources: Consume events from external sources and forward to channels
    * Interceptors: Allow events to be intercepted and modified in flight
    * Selectors: Provide routing for events
    * Channels: Store events until they’re consumed by a sink
    * Sink: Remove events from a channel and deliver to a destination
    * Flume agent: JVM process hosting a set of Flume sources, sinks, channels... Container for components
    
Flume is reliable, recoverable (events are persisted), declarative (no code required) and highly customizable.

##### Flume patterns

    * Fan-in: Several flume agents (web servers or other), which send events to several but fewer agents on Hadoop edge 
    nodes on the same network than the hadoop cluster, which puts the data in HDFS
    * Splitting data on ingest: split events for ingestion into multiple targets, intended for disaster recovery (DR)
    * Partitioning data on ingest: For example, the HDFS sink can partition events by timestamp
    * Splitting events for streaming analytics: sending to a streaming analytics engine such as Storm or Spark Streaming where real-time counts, windowing, and summaries can be made
    
##### File formats:

    * Text files: Not optimal, in general, when you’re ingesting data through Flume it’s recommended to either save 
    to SequenceFiles, which is the default for the HDFS sink, or save as Avro
    * Columnar formats: Columnar file formats such as RCFile, ORC, or Parquet are also not well suited for Flume as 
    they require batching events, which means you can lose more data if there’s a problem
    
Writing to these different formats is done through Flume event serializers, creating custom event serializers is a 
com‐ mon task, since it’s often required to make the format of the persisted data look different from the Flume event
 that was sent over the wire.
 
##### Recommendations
For flume sources:

    * Batch size: Start with 1,000 events in a batch and adjust up or down from there based on performance
    * Threads: In general, more threads are good up to the point at which your network or CPUs are maxed out, but it 
    also depends on the type of source (RDBMS, Avro...)
    
For flume sinks:

    * Number of sinks: A sink can only fetch data from a single channel, but a channel supports multiple sinks
    * Batch Sizes: buffering adds significant benefits in terms of throughput
    
For channels:

    * Memory channels: Use it when performance is your primary consideration, and data loss is not an issue
    * File channels: File channel persists events to disk, configuring a file channel to use multiple disks will help
     to improve performance, using an enterprise storage system such as NAS can provide guarantees against data loss 
     and use dual checkpoint directories
     
To size the channels consider:

    * Memory channels: Consider limiting the number of memory channels on a single node
    * File channels: When the file channel only supported writing to one drive, it made sense to configure multiple 
    file channels to take advantage of more disks
    
##### Finding Flume bottlenecks

    * Latency between nodes: Every client-to-source batch commit requires a full round-trip over the network. Look at
     batch sizes
    * Throughput between nodes: Consider using compression with Flume to increase throughput
    * Number of threads: Adding threads might lead to an improvement
    * Number of sinks: Consider writing with more than one sink to increase performance
    * Channel: Consider channel specific drawbacks
    * Garbage collection issues: Can happen when event objects are stored in the channel for too long
    
### Kafka
Apache Kafka is a distributed publish-subscribe messaging system. 

    * Kafka can be used in place of a traditional message broker or message queue in an application architecture in 
    order to decouple services
    * Kafka’s most common use case is high rate activity streams, such as website clickstream, metrics, and logging
    * Another common use case is stream data processing, where Kafka can be used as both the source of the information 
    stream and the target where a streaming job records its results for consumption by other systems
    
#### Kafka and Hadoop
A common question is whether to use Kafka or Flume for ingest of log data or other streaming data sources into Hadoop:

    * Flume is a more complete Hadoop ingest solution, support for writing data to Hadoop, solves many common issues in 
    writing data to HDFS, such as reliability, optimal file sizes, file formats, updating metadata, and partitioning
    * Kafka is a good fit as a reliable, highly available, high performance data source
    
Flume and kafka are complementary, and there is a flume kafka sink, channel and source.

### Data Extraction
The following are some common scenarios and considerations for extracting data from Hadoop:

    * Moving data from Hadoop to an RDBMS or data warehouse: Sqoop will be the appropriate choice for ingesting the 
    transformed data into the target database. However, if Sqoop is not an option, using a simple file extract from 
    Hadoop and then using a vendor-specific ingest tool is an alternative
    * Exporting for analysis by external applications: A simple file transfer is probably suitable—for example, 
    using the Hadoop fs -get command or one of the mountable HDFS options
    * Moving data between Hadoop clusters: Common in dissaster recovery (DR). DistCp is the solution, which uses 
    MapReduce to perform parallel transfers of large volumes of data. Suitable also when the source or target is a 
    non-HDFS filesystem
    
## Chapter 3: Processing Data in Hadoop<a name="Chapter3"></a>
### MapReduce
#### MapReduce Overview
The MapReduce programming paradigm breaks processing into two basic phases: a map phase and a reduce phase. The 
input and output of each phase are key-value pairs. Data locality is an important principle of MapReduce, when the 
mapper has processed the input data it will output a key-value pair to the next phase, the sort and shuffle (sort and
 partitioning). The reducer will write out some amount of the data or aggregate to a store.
The output of the mapper and the reducer is written to disk. If the output of the reducer requires additional processing
 then the entire data set will be written to disk and then read again. This pattern is called _synchronization barrier_. 
Components involved in the map phase of a MapReduce job:

    * InputFormat: Class used to access the data in the mappers. Implements a method getSplits() which implements 
    the logic of how input will be distributed between the map processes and the getReader() which allows the mapper to 
    access the data it will process
    * RecordReader: Class that reads the data blocks and returns key-value records to the map task
    * Mapper.setup(): Method used to initialize variables and file handles that will later get used in the map process
    * Mapper.map(): This method has three inputs: key, value, and a context. The key and value are provided by the 
    RecordReader and contain the data that the map() method should process
    * Partitioner: Implements the logic of how data is partitioned between the reducers
    * Mapper.cleanup(): Called after the map() method has executed for all records
    * Combiner: Provides an easy method to reduce the amount of network traffic between the mappers and reducers, it
     executes locally on the same node where the mapper executes

There are a few components of which you should be aware on the reduce phase:
    
    * Reducer.setup(): Executes before the reducer starts and is typically used to initialize variables and file handles
    * Reducer.reduce(): Method where the reducer does most of the data processing. The keys are sorted, the value 
    parameter has changed to values
    * Reducer.cleanup(): Called after all the records are processed
    * OutputFormat: When the reducer calls context.write(Km,V), it sends the output to the outputFileFormat, which 
    is responsible for formatting and writing the output data. A single reducer will always write a single file, so 
    on HDFS you will get one file per reducer
    
#### When to Use MapReduce
There is a subset of problems, such as file compaction, distributed file-copy, or row-level data validation, which 
translates to MapReduce quite naturally.

### Spark
The MapReduce model is useful for large-scale data  processing, but it is limited to a very rigid data flow model 
that is unsuitable for many applications. Spark addresses many of the shortcomings in the MapReduce model.

#### Spark Overview
##### DAG Model
Spark allows you to string together sets of map and reduce tasks (these chains are known as directed acyclic graphs).
The spark's engine creates those complex chains of steps from the application’s logic, rather than the DAG being 
 an abstraction added externally to the model.
 
#### Overview of Spark Components
Spark Components:
    
    * Driver: program that defines the logic and the resilient distributed datasets (RDDs) and their transformations
    * DAG scheduler: Queue planner that receives the parallel operations and plans accordingly
    * Cluster manager: has information about the workers, assigned threads, and location of data blocks. Assigns 
    specific processing tasks to workers 
    * Worker: receives units of work and data to manage and send the results back to the driver
    
#### Basic Spark Concepts
##### Resilient Distributed Datasets
RDDs are collections of serializable elements, and such a collection may be parti‐ tioned, in which case it is stored
 on multiple nodes. RDDs store their lineage, so if the data is lost, Spark will replay the lineage to rebuild the lost 
 RDDs so the job can continue.
 
##### Shared variables
Broadcast variables are sent to all the remote execution nodes, accumulators are also sent to the remote execution 
nodes, but they can be modified by the executors.

##### SparkContext
Object that represents the connection to a Spark cluster

##### Transformations
Lazy functions that takes an RDD and returns another. Examples are _map_, _filter_, _keyBy_ (takes an RDD and 
returns a key-value pair RDD), _join_, _groupByKey_ or _sort_.

##### Action
Actions are methods that take an RDD, perform a computation, and return the result to the driver application.

#### Benefits of Using Spark
Benefits or using spark includes:

     * Simplicity: Cleaner API than MapReduce
     * Versatility: Spark was designed and built to be an extensible, general-purpose parallel processing framework
     * Reduced disk I/O: RDDs can be stored in memory and processed in multiple steps or iterations without adding I/O
     * Storage: Options include in memory on a single node, in memory replicated to multiple nodes, or persisted to disk
     * Multilanguage:  Spark APIs are implemented for Java, Scala, and Python
     * Resource manager independence: supports both YARN and Mesos as resource managers, and standalone mode
     * Interactive shell (REPL): Spark includes a shell (REPL for read-eval-print-loop) for interactive experimentation
     
#### When to Use Spark
Spark is the best choice for machine-learning applications due to its ability to perform iterative operations on data 
cached in memory. It also offers SQL, graph processing, and streaming frameworks.

### Abstractions
A number of projects have been developed with the goal of making MapReduce easier to use by providing an abstraction 
that hides much of its complexity. These abstractions can be divided in two different programming models: ETL and query.

### Pig
Pig interprets queries written in a Pig-specific workflow language called Pig Latin, which gets compiled for 
execution by an underlying execution engine such as MapReduce. The scripts are first compiled into a logical plan 
and then into a physical plan, which is what gets executed by the underlying engine (only the physical plan changes 
when you use a different execution engine).

#### When to use Pig
Pig is easy to read and understand, a lot of the complexity of MapReduce is removed, scripts are small and requires 
no compilation (can be run in the console). It also provides explanations of the inner execution of pig.

### Crunch
Similar to Pig, Crunch centers on the Pipeline object (_MRPipeline_ or _SparkPipeline_ which allows you to create your 
first _PCollections_. PCollections and PTables in Crunch play a very similar role to RDDs in Spark and relations in 
Pig. The execution of a Crunch pipeline occurs with a call to the done() method (nothing happens until this method is
 called).
 
#### When to Use Crunch
Similar to Spark, so use Spark instead.

### Cascading
Somewhat of a middle ground between Crunch and Pig

#### When to Use Cascading
Similar to Spark, so use Spark instead.

### Hive
Enables data analysts to analyze data in Hadoop by using SQL syntax without having to learn how to write MapReduce.

#### Hive Overview
Hive is widely adopted, the biggest drawback is performance. To solve performance problems, there are some solutions:

    * Hive-on-Tez: Tez is a more performant batch engine than MapReduce to be used as Hive’s underlying execution engine
    * Hive-on-Spark: Project to allow Spark to be Hive’s underlying execution engine
    * Vectorized query execution: Effort to reduce the CPU overhead required by Hive queries by processing 
    a batch of rows at a time and reducing the number of conditional branches when processing these batches (you need
     to store your data in particular formats like ORC and Parquet to take advantage of this support)
     
#### When to Use Hive
The Hive meta‐store, has become the de facto standard for storing metadata in the Hadoop ecosystem. Hive is a good 
choice for queries that can be expressed in SQL, particularly long-running queries where fault tolerance is desirable. 

### Impala
Impala is an open source, low-latency SQL engine on Hadoop inspired by Google’s Dremel paper. Designed to optimize 
latency, its architecture is similar to that of traditional massively parallel processing (MPP) data warehouses and 
does not use map reduce but uses the Hive metastore.

#### Impala Overview
Impala has a shared nothing architecture, which allows for system-level fault tolerance and huge scalability that 
allows Impala to remain performant as the number of users and concurrent queries increases. Impala’s architecture 
includes the Impala daemons (impalad), the catalog service, and the statestore. Impala daemons run on every node in 
the cluster, and each daemon is capable of acting as the query planner, coordinator, and execution engine.

#### Speed-Oriented Design
Several design decisions to reduce Impala’s query latency compared to other SQL-in-Hadoop solutions were taken:

    * Efficient use of memory: Data is read, and remains in memory as it goes through multiple phases of processing. 
    If you lose a node while a query is running, your query will fail. Therefore, Impala is recommended for queries 
    that run quickly enough that restarting the entire query in case of a failure is not a major event
    * Long running daemons: Impala daemons are long-running processes. There is no startup cost incurred and no 
    moving of JARs over the network or loading class files when a query is executed, because Impala is always running
    * Efficient execution engine: Implemented in C++, highly efficient code, no Java’s garbage collection impact
    * Use of LLVM: Use of Low Level Virtual Machine (LLVM) to compile the query and functions in optimized machine code
    
#### When to Use Impala
Use Impala instead of Hive where possible to make use of the higher speed (hundreds of users who will need to run SQL
 queries concurrently). If the query takes long time to execute use Hive for the risk of node failures.
 
 
## Chapter 4: Common Hadoop Processing Patterns <a name="Chapter4"></a> 
