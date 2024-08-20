
#### Database Workloads

##### OLTP: Online Transaction Processing
fast, short running repetitive operations and simple queries that operate on a single entity at a time. Typically more writes than reads and only read/update a small amount
of data each time
Example: Amazon Storefront; users can add things to their cart and make purchases
but the actions affect only their account
##### OLAP: Online Analytical Processing
long running, complex queries and reads on large portions of the database. In OLAP workloads the database systeem is often analyzing and deriving new data from existing
data collected on the OLTP side.
Example: Amazon computing the most bought item in Pittsburgh on a day when it's raining

##### HTAP: Hybdrid Transaction + Analytical Processing
A new type of workload where OLTP and OLAP are present together on the same database

#### Storage Models
##### N-Ary Stoarage Model (NSM) - (Row store)
Stores all of the attributes for a single tuple contigously in a single page.
Ideal for OLTP where requests are insert-heavy and transactions tend to operate only
an individual entity, since it takes only one fetch to be able to get all of the attributes for a single tuple 
Advantages:
- Fast insersts, updates, and deletes.
- Good for queries that need the entire tuple.
Disadvantages:
- Inefficient for scanning large portions of the table and/or a subset of the attributes.

##### Decomposition Storage Model (DSM) - (Column store)
Stores a single attribute(column) for all tuples contigously in a block of data.
Ideal for OLAP with many read-only queries that perform large scans over a subset
of the table's attributes.
Advantages:
- Reduces the amount of I/O wasted because the DBMS only reads the data that it needs for that query
- Better query processing because of increased locality and cached data reuse
- Better data compression
Disadvantages:
- Slow for point queries, inserts, updates and deletes because of tuple splitting
/stitching
There are two common approaches to put the tuples back together:
- Most commonly used apptoach is fixed-length offsets. Here the value in a given column will belong to the same tuple as the value in another column at the same offset. Therefore every single value within the column will have to be the same length
- Less common approach is to use embedded tuple ids. Here for every attribute in columns, the DBMS stores a tuple id(ex: primary key) with it. Then the system would also sotre a mapping to tell it how to jump to every attribute that has that id. Note that this method has a larga storage overhead because it needs to store a tuple id for every attribute entry

##### Partition Attributes Across (PAX) - (Hybrid)
Rows are horizontally partitioned into groups of rows. Within each row group, the attributes are vertically partitioned into columns. Each row group is similar to a column store for its for its subset of the rows -> benefit of faster processing on columner storage while retaining the spatial locality benefits of row storage
a PAX file has a global header containing a directory with offsets to the file's
row groups and each row group maintains its own header with metadata about its
contents

#### Database Compression
Widely used in disk based DBMS as I/O is (almost) always the main bottleneck,
especially popular in systems that have read-only analytical workloads

tradeoff between speed vs compression ratio for usecase:
io speed(size) + decomp(ratio) + ?network transmission speed(size) = eta to usabillity

If data sets were completely random bits, there would be no way to perform compression but there are key properties of real world data sets that are amenable to compression:
- Highly skewed distributions for attribute values (Zipfian distribution of the Brown Corpus)
- High correlation between attributes of the same tuple (Zip Code to City, Order Date to Ship Date)
Given this we want a database compression scheme to have the following properties
- Must produce fixed-length values. The only exception is variable length data stored in seperate pools. This is because the DBMS should follow word-alignment and be able to access data using offsets
- Allow the DBMS to postpone decompression as long as possible during query execution (late materialization)
- Must be a lossless scheme because people do not like losing data. Any kind of lossy compression has to be performed at the application level

##### Compression Granularity
- Block level: Compress a block of tuples for the same table
- Tuple Level: Compress the contents of the entire tuple (NSM only)
- Attribute Level: Compress a single attribute value within one tuple. Can target multiple attributes for the same tuple
- Columnar Level: Compress multiple values for one or more attributes stored for multiple tuples (DSM only). This allows for more complicated compression schemes

##### Naive Compression
Compress data using a general purpose algorithm
- LZO (1996)
- LZ4 (2011)
- Snappy (2011)
- Brotli (2013)
- Zstd (2015)
Example: MySQL InnoDB compresses disk pages, pads them to a power of 2KBs and stores them into the buffer pool. Howerver every time the DBMS tries to read/modify data the compresssed data in the buffer pool must be first decompressed.

Since accessing data requires decompression of compressed data, this limits the scope of the compression scheme. If the goal is to compress the entire table into one giant block, using naive compression schemes would be impossible since the whole table needs to be compressed/decompressed for every access. Therefore, mysql breaks the table into smaller chinks since the compression scope is limited

Another problem is that these naive schemes also do not consider the high-level meaninf or semantics of the data. The algorithm is oblivious to both the structure of the data, and how the query is planning to acces the data. This this eliminates the opportunity to utilize late materialization, since the DBMS cannot tell when it can delay the decompression of data

##### Columnar Compression

###### Run Length Encoding
Compresses runs(consecutive instances) of the same value in a single column into triplets of (Value, Offset, #count)
Some preprocessing steps or chaining columnar compression methods yield even better results e.g sorting a true false table then RLE -> (True, [range]) (False, [range]) fits into only 2 rows
Sometimes called null supression if DBMS only tracks empty space
###### Bit-Packing Encoding
When all values for an attribute are less than the values declared largest size, store them with fewer bits
###### Mostly Encoding
When there are big outliers in the dataset but the rest fit in a smaller size value
use bit-packing and map/store the outliers somewhere else 
###### Bitmap Encoding
Bitmap Index Compression:
1. General Purpose Compression
2. Byte-aligned Bitmap Codes
3. Roaring Bitmaps
###### Delta Encoding
track changes instead of each big value, combine with rle for better compression
###### Dictionary Encoding
Most common database compression scheme is dictionary encoding. The DBMS replaces frquent patterns in values with smaller codes, it then stores only these codes and a data structure(dictionary) that maps these codes to their original value. A dictionary compression scheme needs to support fast encoding decoding as well as range queries

- When to construtct the dictionary?
    1. All-At-Once:
        - Compute dictionary for all the tuples at a given point of time
        - New tuples must use a seperate dictionary, or all tuples must be recomputed
        - easy to do if the file is immutable
    2. Incremental:
        - Merge new tuples in with an existing dictionary
        - Likely requires re-encoding to existing tuples
- What is the scope of the dictionary?
    1. Block-level:
        - Only include a subset of tuples within a single table
        - DBMS must decompress data when combining tuples from different blocks
    2. Table-level:
        - Construct a dictioinary for the entire table.
        - Better compression ratio, but expensive to update
    3. Multi-Table:
        - Can be either subset or entire tables
        - Sometimes helps with joins and set operations
- What data structure do we use for the dictionary?
    1. Array:
        - One array of variable length string and another array with pointers that maps to string offsets
        - Expensive to update so only usable in immutable files
    2. Hash Table
        - Fast and compact
        - Unable to support range and prefix queries
    3. B+ Tree
        - Slower than a hash table and takes more memory
        - Can support range and prefix queries enabling smart query optimizations
- What encoding scheme to use for the dictionary
Should be order preserving; encoded values need to support sorting in the same order as original values

Parquet / Apache ORC do not provide an API to directly access a file's compression dictionary. This means the DBMS cannot perform predicate pushdown and operate directly on compressed data before decompressing it

#### DBDB.io study

compression methods
- Bitmap Encoding
- Bit Packing / Mostly Encoding
- Delta Encoding
- Dictionary Encoding
- Incremental Encoding
- Naive (Page-Level)
- Naive (Record-Level)
- Null Suppression
- Prefix Compression
- Run-Length Encoding (RLE)

storage formats:
- Apache Arrow
- Apache Avro
- Apache CarbonData
- Apache Hudi
- Apache Iceberg
- Apache ORC
- Apache Parquet
- CSV
- Custom
- DataFrame
- HDF5
- N-Triples
- RCFile
- SequenceFile
- Trevni

Checkpoints
- Blocking
- Consistent
- Fuzzy
- Non-Blocking
- Not Supported

huffman encoding is kinda like bpe but outputting bits?


##### clickhouse:
SQL support
Table oriented
Faster aggregations; 5x
Better compression; %90
Column-oriented database
Ordering key, data types, codecs can be defined in schema enabling us to exploit better compression rate through various properties of the dataset

#####elasticsearch:
compression:
- LZ4: faster compresssion and decompression, lower comp ratio; 780/4970 : 2.101
- DEFLATE: slower compressiond and decompression, higher comp ratio; 100/415 : 2.730
Schemaless : Document oriented
Custom DSL query lang
Full-text search



best performing BPE tokenizers seem to be doing (tiktoken/huggingface) 40-70mb/s max
is it cause of the large tokenizer dictionaries?

sources:
- https://duckdb.org/2022/10/28/lightweight-compression.html
- https://duckdb.org/docs/internals/storage.html
- https://duckdb.org/docs/guides/performance/file_formats.html
- https://rocksdb.org/blog/2021/05/31/dictionary-compression.html
- https://developer.chrome.com/blog/shared-dictionary-compression
-
- https://www.elastic.co/guide/en/elasticsearch/reference/7.17/tune-for-disk-usage.html
-
- https://github.com/facebook/zstd#the-case-for-small-data-compression
- https://groups.google.com/g/redis-db/c/slk-c33EZ7U/m/tx81gCMDDQAJ
- https://docs.rs/zstd/latest/zstd/stream/index.html
- https://docs.rs/zstd/latest/zstd/dict/index.html
- https://facebook.github.io/zstd/zstd_manual.html
-
- 
- https://parquet.apache.org/docs/file-format/data-pages/compression/
- 
- https://www.youtube.com/watch?v=UuHw3SQMrog [F2023 #05 - Storage Models & Database Compression (CMU Intro to Database Systems)]
- https://www.youtube.com/watch?v=Bpy5hYyHAuQ [05 - Database Compression (CMU Advanced Databases / Spring 2023)]
-
- https://clickhouse.com/docs/en/concepts/why-clickhouse-is-so-fast
- https://clickhouse.com/docs/en/data-compression/compression-in-clickhouse
- https://clickhouse.com/blog/optimize-clickhouse-codecs-compression-schema
- https://clickhouse.com/docs/en/sql-reference/statements/create/table#specialized-codecs