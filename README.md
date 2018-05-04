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

### Timeliness of Data Ingestion
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

### Incremental Updates
HDFS has high read and write rates because of its ability to parallelize I/O to multiple drives. The downside to HDFS
is the inability to do appends or random writes to files after they’re created. Consider the following:
 
    * Create periodic process to join multiple small files, using a long consecutive scan to read a single file is 
    faster than performing many seeks to read the same information from multiple files
    * To update files, try to write a “delta” file that includes the changes that need to be made to the 
    existing file and create a compaction job to handle modifications, or consider HBase for this
    
In a compaction job, the data is sorted by a primary key. If the row is found twice, then the data from the newer 
delta file is kept and the data from the older file is not. The results of the compaction process are written to 
disk, and when the process is complete the resulting compaction data will replace the older, uncompacted data.

### Access Patterns
Recommendation for cases where simplicity, best compression, and highest scan rate are called for, HDFS is the 
default selection. When random access is of primary importance, HBase should be the default, and for search 
processing you should consider Solr.

### Original Source System and Data Structure

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
    
### Transformations
Transformation refers to making modifications on incoming data, distributing the data into partitions or buckets, or 
sending the data to more than one store or location. The same advices holds here batch when the timeliness is not 
that much of a concern and kafka, flume, spark for stream. Configure two output directories: one for records that 
were processed successfully and one for failures. Flume has interceptors (Java class that allows for in-flight 
enrichment of event data) and selectors (to send events to different endpoints) to deal with this problem.

### Network Bottlenecks
If the network is the bottle‐ neck, it will be important to either add more network bandwidth or compress the data 
over the wire. 

### Network Security
Encrypt data on the wire if it needs to reach outside the company network boundaries (Flume provides native support).

### Push or Pull
