# Designing Data-Intensive Applications

[TOC]



# PART I. Foundations of Data Systems

## Chapter 1. Reliable, Scalable, and Maintainable Applications

Many applications today are data-intensive, as opposed to compute-intensive

### Reliability

Fault and failure: System need to be fault-tolerance

- Hardware Errors

  1. hard disks fails -> add redundancy (reduce rate) -> larger data requires larger number of machine (increase rate)  -> more virtual machines (increase rate)
  2. Such tolerance also allows rolling upgrade (patch one node a time, no downtime)

- Software Errors

  - More correlated across nodes

  - Often because it made assumptions and that assumptions was no longer true

  Solutions

  - (Before launch) Testing, in design, isolate process and allows crash
  - (After launch) monitoring, analyzing, alerting

- Human Errors

  - Config errors by human is the leading cause of outages

  Solutions

  - System: well-design abstraction
  - Decouple the place of mistakes: use sandbox
  - Test
  - Quick and easy recovery. Allows recomputing data
  - Monitoring
  - Management and training

### Scalability

  It's not a label: "X is scalable" or "Y doesn't scale". But rather "If the system grows in a particular way, what are the options"
  **Describing Load**

  - *Load parameter*
    - e.g. requests per second, ration of reads to writes, hit rate on cache

  - Twitter example

    Approach 1: user's timeline from query
    Approach 2: cache for each user and inserting on new tweet
    Twitter moved from 1 to 2, then celebrities with many followers got slow.
    Solution: celebrities use Approach 1

    In this example, the distribution of followers per user is a key load parameter for scalability

  **Describing Performance**

- batch processing - throughput; service - response time

- Latency vs response time: 
  response time = queueing + processing + network delay 
  Latency = network delay (maybe plus queueing?)

- Average isn't good enough, median or percentiles is better. the 95th, 99th, and 99.9th percentiles are common (abbreviated p95, p99, and p999).

- Tail latencies are important
  e.g. Amazon use 99.9th percentile, since slowest requests are often with the most data and are often most valuable customers
- How to measure
  - Measure response time on client side
    A few slow requests can delay the whole queue, even those subsequent request are fast *"head-of-line blocking"*
  - Keep sending request, rather than wait for response and then send
  - *"tail latency amplification"*: one user requests takes several backend requests, the slowest backend request slows down the entire user request
  - rolling window metrics: many ways to calculate a good approximation: forward decay, t-digest, or HdrHistogram. Besides, for multiple machine, avg percentile doesn't make sense, need to aggregate histograms.

**Approaches for Coping with Load**

vertical scaling (more powerful machine) 
horizontal scaling (multiple machines, aka *shared-nothing* architecture)

Elastic: automatic
scales manually: less surprise

Advices:

- Stateful data is harder to distribute. Single node first until not viable

- It is conceivable that distributed data systems will become the default in the future, even for use cases that don’t handle large volumes of data or traffic. But not yet!

- The architecture of systems that operate at large scale is usually highly specific to the application. No one-size-fits-all scalable architecture *"magic scaling sauce"*

  volume of reads / writes / data to store / complexity of the data / response time / access pattern

- Depends on load parameter assumption. If they are wrong, wasted or counterproductive
  Early stage, future load are still hypothetical, product feature quick iteration is usually more important

### Maintainability

**Operability**: Making Life Easy for Operations

Anticipating future, better definition and knowledge sharing.
Visibility into runtime and system. Good default as well as admin freedom

**Simplicity**: Managing Complexity

One of the best tools we have for removing *accidental complexity* is *abstraction.*

**Evolvability**: Making Change Easy
Agile, test-driven development


## Chapter 2. Data Models and Query Languages
### Relational Model Versus Document Model
NoSQL: why born? Because scalability / open source over commercial / specialized query / flexible schema and expressive data model

*polyglot persistence*

*impedance mismatch* 

DB <---mismatch---> OOD. ORM (Object-relational mapping) helps reduce

**Example: Resume**
It has many one-to-many relationships
Option 1: traditional SQL model (before1999) separate tables
Option 2: XML or JSON data type; multi values in single row and querying/indexing
Option 3: encoded as XML/JSON but stored in txt (downside: can not use db to query)

This data structure is mostly a self-contained *document*, JSON representation can be quite appropriate, reduces the impedance mismatch

The JSON representation has better locality than the multi-table schema. (One query, no join)
JSON representation makes this tree structure explicit。

#### Many-to-One and Many-to-Many Relationships

Benefits of standardized lists of geographic regions and drop-down list for user

1. Consistent style and spelling
2. Avoiding ambiguity (e.g.same name)
3. Ease of updating
4. Localization support (language)
5. Better search

Stores ID VS Stores text

- Use ID
  human-meaningful information stored in one place. 
  ID doesn't need to change. *"Normalization"*
- Use text:
  human-meaningful information duplicating. 
  All duplicated copies need to be updated
  Risk: write overheads and inconsistencies

Relational DB support joins. Document DB, no need for one-to-many, support for joins is weak

If DB doesn't support, need application supports. (slow-changing, small, can fit into memory) Still the work is shifted from DB to App.

Besides data has a tendency of becoming more interconnected

#### Are Document Databases Repeating History?

*Hierarchical model* -> *relational model* and *network model*

**Network model**

Allows multiple parents. No foreign key. Like pointers in programming language
Only way of accessing record is *access path* (like linked list)

- Flexibility?
  Need to keep track of the access path:
  Change to access path would need updating a lot of queries

**Relational model**

Can read row directly
Query optimizer decides the execution plan (like the access path) but automatically.

- Flexibility?
  If you want to query in new way, only need to add new index and they will be used automatically. Thus easier to add new features

Query optimizers are complicated beasts. But it only need to build once and all who use the database can benefit (harder, but general-purpose solution wins in the long run)

**Comparison to document databases**

Document DB, like hierarchical, stores one-to-many
But for many-to-one and many-to-many, relational and document DB are quite similar, they both use *foreign key* (called *document reference* in document model

#### Relational Versus Document Databases Today

**Which leads to simple application code?**

Document model

Pros

- no shredding (split to multiple tables in relational db)

Cons

- Can't refer directly to nested item (thus better not too deep)
- Poor join (but not needed, e.g. in analytics app)

In many-to-many case, since you can't join, application has to renormalize, and need to keep data consistent, making it more complex and slow.

interconnection

| Interconnection | low      | high       | very high |
| --------------- | -------- | ---------- | --------- |
| suggestion      | document | relational | graph     |

**Schema flexibility in the document model**

*No schema. schemaless*. But it's misleading. 

| Document | Relational  |
| --------------- | -------- |
| schema-on-read  | schema-on-write  |
| Like dynamic type checking | Like static type checking |

Schema change is not slow or have downtime: most DB do ALTER TABLE quick, except MySQL copies entire table.

When to use?

- Many different type of objects. Don't want each to be a table
- Structure determined by external system. Can't control the change

**Data locality for queries**

Faster retriving
But need to keep each document small (and not increase)

(For two reasons, 1. db loads the entire document 2. changes rewrite the entire document, unless size not change then it can be in-place)

Relational DB can be localized too (parent table, multi-table index cluster, column-family)

**Convergence of document and relational databases**

Relational supports XML, JSON

Document supports join

### Query Languages for Data

|                   | imperative query                      | declarative query                                            |
| ----------------- | ------------------------------------- | ------------------------------------------------------------ |
| Def.              | How to do it step by step             | What's the pattern of the result                             |
| Example           | Gremlin                               | SQL, css, cypher                                             |
| Execution         | strictly defined                      | hide execution details: DB can improve without changing the queries |
|                   | Changing ordering may break the query | Order of data is not guaranteed and doesn't matter           |
| Parallelizability | Hard                                  | Easier                                                       |

**Declarative Queries on the Web**

Use css as an example. If it were imperative, besides it;s harder, there are serious problem

1. A little change, the browser should automatically detect it's no longer match the rule, but imperative can not achieve that
2. If you want to use new API that improves performance, you need to rewrite the code. If declarative browser can do it without breaking comparibility

**MapReduce Querying**

MapReduce is neither declarative query nor fully imperative

Restrictions:

- They must be pure functions
  - no database queries
  - no side effects

Thus they can be run in any order, and rerun on failure

Notice: SQL can distribute. MapReduce can single machine. Some SQL can be distributed without using MapReduce. Some SQL DB can run JavaScript function too

Usability problem:
Writing those two coordinated functions is harder than a single query

(MongoDB supported declarative query language, JSON-based syntax, but may accidentally reinventing SQL)



### Graph-Like Data Models

Use cases: social network, web network, road or rail networks

Algorithms: shortest path, PageRank for web graph

Types

- *property graph* model (Neo4j, Titan)
- *triple-store* model (Datomic, AllegroGraph)

#### Property Graphs

- Vertex: UID, outgoing edge, incoming edge, properties (K-V pairs)
- Edge: UID, tail vertex, head vertex, relationship, properties (K-V pairs)

Can be represented in a relational schema

Good flexibility. for example:
France uses region US use states
Different granularity, when an item sometime specified as city, sometime as states

#### The Cypher Query Language

Declarative

Executing plan can be various

#### Graph Queries in SQL

Can we query graph like SQL? Yes but difficult

Because the number of joins is not fixed. You don't know which table to join in advance

(SQL: use WITH RECURSIVE, 29 lines, Cypher: 4 lines)

#### Triple-Stores and SPARQL

*(subject, predicate, object)* e.g. `(Jim, likes, bananas)`

- `(lucy, age, 33)` like the vertex with property

- `(lucy, marriedTo, alain)` likes the vertex - edge - vertex

*Turtle format*

**The semantic web and RDF**

Triple-store is independent of the semantic web

"semantic web": website publish info as machine-readable data: Resource Description Framework "RDF". Overhyped in the early 2000s

RDF: use URI to avoid conflicts. 
(Design for combining with someone else’s data, same word “within” may refer to different things)

**The SPARQL query language**

Predates Cypher
RDF doesn’t distinguish between properties and edges but just uses predicates for both

**Graph Databases Compared to the Network Model**

Are Graph DB CODASYL in disguise? No

- CODASYL has schema. Graph doesn't
- CODASYL, only way to reach record was to traverse one path. Graph has UID
- CODASYL children of record is ordered. Graph is not
- CODASYL, all queries were imperative. Graph supports declarative query like Cypher, SPARQL

#### The Foundation: Datalog
Datalog is much older, provides the foundation that later query languages build upon.
*predicate(subject, object)*

Rules, like function calls, can be recursive, can be combined and reused

### Summary

History - hierarchical model

SQL - Relational model

NoSQL

- Document model
- Graph model

But there are still many other data models

- Genome: need sequence similarity searches on very  long string (DNA molecule). Similar but identical. Genome DB like GenBank
- Particle Physicists: Large Hadron Collider (LHC) hundreds of petabytes. Custom solutions are required for controllable hardware cost
- Full-text search

## Chapter 3: Storage and Retrieval

Two families of storage engines: *log-structured* storage engines, and *page-oriented* storage engines such as B-trees.

### Data Structures That Power Your Database

Simplist: key value store with plain text, or value can be JSON

```
123456 '{"name":"London","attractions":["Big Ben","London Eye"]}'
```

(Real database use a *log* too, just with concurrency control and so on. Different from *application log*, this is more general:"append-only sequence of records")

- Good performance on write. Appending file is efficient

- Bad performance on read. O(n)

**Index**: Important trade-off

- Additional structure. Doesn’t affect the contents, only affects performance of queries. 
- Incurs overhead, especially on writes. 

#### Hash Indexes

| --              | Design                                                       | How to query                                                 | Advantage                                                    | Limitation              | Use Case                                                     |
| --------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ----------------------- | ------------------------------------------------------------ |
| Simplist        | A hashmap in memory for offset of the data (Bitcask uses this) | find key in memory                                           | Quick look up                                                | All keys fit in the RAM | e.g. key is URL of video, value is number of time. Large write per key, not too many keys |
| With compaction | Break the log into segments, then compact (throw away duplicate keys, keep latest) | One hash map for each segment. Search key in map starting from the most recent segment | Can merge several segments. Can be done in a background thread, while serve read/write using old, then switch |                         |                                                              |

Improvements

- File format
  cvs not good. `<length of string in binary> + raw string` no need to escape
- Deleting records
  append a deletion record: tombstone
- Crash recovery: in-memory hash maps lost
  Too slow to rebuild map from segments. Store a snapshot of each hash map on disk
- Partially written record
  Detect with Checksums
- Concurrency control
  Common way: have only one writer thread.

**Append-only log Pros and Cons**

| Pros                                                         | Cons                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Appending, segment merging are **sequential write operations**. **Faster** especially on magnetic spinning-disk hard drives. (Also preferable on SSD) | **Large key not good**. Difficult to make an on-disk hash map perform well |
| **Concurrency and crash recovery** are much **simpler** since segment files are append-only or immutable. (file would not contain old and new value spliced together) | **Range queries are not efficient.** For e.g. Cannot easily scan over all keys between kitty00000 and kitty99999 |
| Merging old segments **avoids** the problem of **data files getting fragmented** over time |                                                              |

#### SSTables and LSM-Trees

Sorted String Table

- key-value pairs is sorted by key
- each key only appears once within each merged segment file

Benefits:

1. Merging: simple and efficient (*mergesort*, doesn't have to fit in memory)
2. Finding key is easier: index can be sparse, skip some keys
3. Can group and compress before writing into disk. Saving disk, reduce I/O bandwidth (since you need to scan several keys anyway due to 2)

**Constructing and maintaining SSTables** 

How to get data sorted in the first place?

- Maintaining a sorted structure on disk is possible: B-Trees
- Maintaining a sorted structure in memory is easy: red-black tree, AVL tree

Steps

1. write to in-memory balanced tree (called *memtable*)
2. Bigger than threshold -> write to disk
3. Read: check memtable, then disk segment from latest to next older
4. Run a merging and compaction process in backgroung

One problem: DB crashes. Thus a separate append-only log

**LSM-Tree: Log-Structured Merge-Tree** 

memory buffer + several levels of SSTables, merged in the background

![img](ddia-reading-note.assets\LSM-Trees_Structure.jpg)

Write operation:

1. WAL LOG
2. Memtable (ensure orderliness, can be a black-red tree or jump table)
3. Transformed into an immutable memtable. Another new memtable
4. Immutable memtable merged and flashed to disk



Full-text search engine use similar method: a term(a word) -> IDs of documents

**Performance Optimizations**

LSM-tree algorithm can be slow when looking up keys that do not exist
Solution: ***Bloom filters***

Strategies to determine the order and timing of how SSTables are compacted and merged: *size-tiered* and *leveled compaction*

LSM-trees—keeping a cascade of SSTables that are merged in the background—is simple and efficient

- Even when the dataset is much bigger than the available memory
- Can range query
- Remarkably high write throughput, because the disk writes are sequential

#### B-Trees

log-structured indexes - *Segments*
B-trees                            - *fixed-size pages* (~4KB)

Each page has a pointer. *Leaf page* contains the value or references

The number of references to child pages in one page of the B-tree is called the *branching factor.* (typically several hundred)

**Adding a new key:** If no enough space, split into two, and parent page also updates

Ensures the tree remains *balanced*: `depth = O(log n) `

Most DB: B-tree only 3 or 4 levels deep

>  A four-level tree of 4 KB pages with a branching factor of 500 can store up to 256 TB
>
> (500^4)*4KB = 2500 *10^8 = 250TB

**Making B-trees reliable**

B-trees modify files in place

Problem 1: On SSD must rewrite large block. Sometimes **several pages** (dangerous if crashes, corrupted index, orphan page)

Solution: write-ahead log (**WAL**, also known as a redo log): an append-only file

Problem 2: **concurrency control**

Solution: ***latches*** (lightweight locks)
Logstructured approaches are simpler in this regard, because they do all the merging in the background without interfering with incoming queries

**B-tree optimizations**

- Rather than WAL, some DB use **copy-on-write** scheme. Modified page written elsewhere, new version of parent page point to it (good for concurrency)

- Save space by **not storing entire key**, only abbriviation. (higher branching factor)

- **"nearby key ranges to be nearby on disk"** not required, but good for key range scan read. Many B-trees implement, but hard to maintain. LSM easiler to maintain since it rewrites during merge.

- Additional pointer: **reference to sibling pages**, also help scanning

- B-tree variants *"fractal trees"* borrow some log-structured ideas to reduce disk seeks

  > fractal trees: a write-optimized data structure
  >
  > Why B-tree slow? Only part of the tree fit into memory. Every write need I/O to find the key
  >
  > B-trees node = pivot & pointer
  >
  > Fractal  node = pivot & pointer & **buffers**
  >
  > - in the root node, find out which child the write SHOULD traverse down
  > - serialize the pending operation into the buffer
  > - if the buffer associated with that child has space, **return**. If the node’s buffer has no space, **flush** the pending operations in the buffer down a level, thereby making space for future writes.
  >
  > Why is it faster? Reduced I/O. Do an I/O for not one row, but many rows
  >
  > Another interesting thing is if everything fits in memory, then Fractal Tree indexes are not providing any algorithmic advantage over B-Trees for write performance.

#### Comparing B-Trees and LSM-Trees

B-Tree:      Faster read    slower write
LSM-Tree: Faster write   slower read

However, benchmarks are often inconclusive and sensitive to details of the workload

**Detailed Comparison**

***write amplification***

B-tree

- write at least twice: to WAL and page (if pages are split write +1)
- write entire page even small change

LSM

- Also rewrite multiple time due to compaction and merging

Particular concern on SSDs (only limited number of writes before wearing out)

**Pros of LSM**

1. lower write amplification (usually)

   B-tree may overwrite several pages. LSM can do sequentially write (specially good for magnetic hard drive)

2. reduced fragmentation
   B-tree fragmentation. LSM removes fragmentation in compaction, lower storage overhead

3. On SSD: many SSD can turn random writes to sequential writes. But 1 + 2 allows more read and write within same I/O bandwith

**Cons of LSM**

1. Compaction can interfere with performance
   Response time mostly small, but at higher percentiles very high. B-trees: more predictable

2. Sharing write bandwidth

   The bigger the DB is, the more disk bandwidth is required for compaction

   Compaction can not keep up with incoming write, many segments, causing 1. disk full 2. slow read

3. Transaction isolation and lock

   B-tree: key exists in one place
   LSM: key exists in multiple files
   Transaction isolation is implemented using locks on ranges of keys
   In a B-tree index, those locks can be directly attached to the tree

#### Other Indexing Structures

secondary indexes: good for join
key are not unique:
Solution 1: key -> [list of row identifiers]
Solution 2: making key unique by appending a row identifier

**Storing values within the index**

Value can be actual row or reference to elsewhere (***heap file***)

1. Only references ***"Nonclustered index"***

  Heap file: Avoids duplicating data when multiple secondary indexes are present
  Update value: If new value is larger and need to move to another place: need to update all index, or forward pointer

2. Row directly within an index **"Clustered index"**
    Read performance better

Between clustered and nonclustered: A compromise: "covering index" / "index with included columns". Can "cover" some of the queries

More duplication -> faster read, overhead on write, addtional effort for transactional guarantees

**Multi-column indexes**

*"concatenated index"*

One usecase: geospatial

```sql
SELECT * FROM restaurants WHERE latitude > 51.4946 AND latitude < 51.5079 AND longitude > -0.1162 AND longitude < -0.1004;
```

A standard B-tree or LSM-tree index is not able to answer that kind of query efficiently

Alternatives

- Translate into a single number using a space-filling curve and then B-tree index
- Use R-trees

Another usecase: RGB Color (red, green, blue) e.g. on an ecommerce website search products by color

**Full-text search and fuzzy indexes**

similar keys,  grammatical variations

Lucene: search text for words within a certain edit distance 

(edit distance of 1 means added/removed/replaced)
SSTable-like term dictionary
Need search within an offset in the sorted file

Index is not sparse, but a finite state automaton over the characters in the keys, similar to a trie. 

Levenshtein automaton

**Keeping everything in memory**

We use disk because 1. durable (contents are not lost if the power is off), and 2. cheaper than RAM

As RAM becomes cheaper, the cost-per-gigabyte argument is eroded.

*inmemory databases*

- acceptable for data to be lost (e.g. *Memcached*)
- achieve durability: special hardware, write log to disk, snapshot to disk, replica 
- Weak durability by writing to disk asynchronously (e.g. Redis)

Counterintuitively, the performance advantage of in-memory databases is not "no read from disk" (Even a disk-based storage engine may never need to read from disk if you have enough memory)
They can be faster because no **overheads of encoding** data for writting to disk

Support more data models. e.g. Redis offers a database-like interface to priority queues and sets

Support datasets larger than memory. 
*"anti-caching approach"* Evicting the least recently used data from memory to disk, and loading it back into memory (still requires indexes to fit entirely in memory)
Similar to what operating systems do with virtual memory and swap files, but the database can manage memory more efficiently, smaller granularity of individual records rather than entire memory pages

Storage engine design may change if non-volatile memory (NVM) technologies become more widely adopted 

### Transaction Processing or Analytics?

Transaction: low-latency reads and writes - as opposed to *batch processing*

Doesn't have to be ACID (atomicity, consis‐ tency, isolation, and durability)

OLTP: online transaction processing

OLAP: online analytic processing

| --              | OLTP                  | OLAP                 |
| --------------- | --------------------- | -------------------- |
| Read            | small records, by key | large number         |
| Write           | random                | Bulk import / stream |
| Used by         | End user via web app  | Interal analyst      |
| Data represents | Latest state of data  | History              |
| Size            | GB/TB                 | TB/PB                |

#### Data Warehousing

Data Warehouse: A separate database for OLAP

ETL: (Extract–Transform–Load) transformed into an analysis-friendly schema, cleaned up, and then loaded into the data warehouse

Advantage of using data warehouse:

- No harm for performance of OLTP
- Can be optimized for analytic access patterns

**Divergence between OLTP databases and data warehouses**

DB vendors focus on supporting only one, but not both

#### Stars and Snowflakes: Schemas for Analytics

(Even date and time are often represented using dimension tables, because this allows additional information about dates (such as public holidays) to be encoded, allowing queries to differentiate between sales on holidays and non-holidays.)

| --       | Star schema                                                  | Snowflake schema                                 |
| -------- | ------------------------------------------------------------ | ------------------------------------------------ |
| def      | fact table + dimentional table                               | Star, but subdomains                             |
| property | simpler to work with.(Join only happens between fact and dimentional table) | more normalized, less redundency. Need more join |

### Column-Oriented Storage

Problem: fact table too wide. Load full row in memory is not necessary

Solution: Store all the values from each column in a separate file

#### Column Compression

Column-Oriented is easy to compress

***bitmap encoding***

***bitmap index compression***

Because number of distinct values **n** in a column is often smaller than total number of rows

Small n:    (like country code) store one bit per row
Larger n:  *run-length encoded*  (avoid too many zeros)

```
column values:
aaa bbb ccc aaa ccc aaa bbb aaa

bitmap
"aaa"  [1, 0, 0, 1, 0, 1, 0, 1]
"bbb"  [0, 1, 0, 0, 0, 0, 1, 0]
"ccc"  [0, 0, 1, 0, 1, 0, 0, 0]


What if there are 1000 unique values?
"aaa" [1, ... 0, 1, 0, ... 0 ]  
(maybe only 0.1% are 1 and 99.9% are zero )

Use run-length encoded
"aaa" -> 0, 1, 800, 1
(means 0 zeros, 1 ones, then 800 zeros, then 1 ones, rest zeros)
```

Bitmap indexes can also help with certain queries, by bitwise operation

example 1: `where product in (30, 68, 69)`
Find bitmap of these three and calculate the OR to target the row

example 2: `where product = 20 and store = 30`
Find bitmap of these two and calculate the AND to target the row

**Column families** 
Cassandra and HBase have a concept of *column families*, but still *row-oriented*
Within each column family, they store all columns from a row together, along with a row key, and no column compression

**Memory bandwidth and vectorized processing**

Big data processing bottlenecks

1. bandwidth for getting data from disk into memory
2. **bandwidth from main memory into the CPU cache**
   - avoiding branch mispredictions and bubbles
   - making use of single-instruction-multi-data (SIMD) instructions in modern CPUs 

Column-oriented storage is also good for making efficient use of CPU cycles, because

1. Query engine can iterate through column data in a tight loop (that is, with no function calls) thus much faster
2. Column compression allows more rows to fit in the CPU cache.
3. *Vectorized processing*: Bitwise operators can be designed to operate oncompressed column data directly

#### Sort Order in Column Storage

Can't sort each column independently.

e.g. date as first sort key, product id as second sort key

Advantages:

1. Optimize the query (no need to scan all)
2. help with compression (run-length compression)
   Compression effect is strongest on the first sort key

**Several different sort orders**

Data needs to be replicated anyway, why not sort in different ways?

Like having multiple secondary indexes in a row-oriented store. But

- Row-oriented store：just pointers
- Column store: Actual values

#### Writing to Column-Oriented Storage

Compressed column: an insertion has to update all columns files consistently

Solution: LSM-trees. In-memory store, later write in bulk

Query need to examine both recent write in memory and column data on disk

#### Aggregation: Data Cubes and Materialized Views

*materialized aggregates.*

If aggregate functions (COUNT, SUM, AVE, MAX, MIN) are used many times, why not cache them?

One way: ***materialized view***

materialized view:  actual copy of result, written to disk
virtual view:             shortcut for queries

update of *materialized view*: If automatic, will make writes slower, thus not in OLTP

 ***data cube*** or *OLAP cube*

E.g. assume the fact table have two dimension tables, date and product.
Then the cube can have

1. sum for each product on each date.
2.  (reduce by one dimention) all product on one date, all date on one product

In general, facts often have more than two dimensions.

Pros: certain query can be very fast
Cons: not flexible, i.e. adding a fillter "sales from items with price > $100"

### Summary

**OLTP vs OLAP**

OLTP: index to find data. Disk seek time is bottleneck
(thus need index)

OLAP: less number of queries, each scan millions of rows. Disk bandwidth is bottleneck 
(thus need compression)

**Storage engine**

OLAP: column oriented

OLTP

- log structured: SSTables, LSM-trees
  Turn random writes to sequential writes: higher throughput
- update-in-place(fixed-size pages): B-tree

## Chapter 4 Encoding and Evolution

*evolvability*

Data format changes. The application code change often can not happen instantaneously

- Server-side: rolling upgrade
- Client-side: at eh mercy of the user

*Backward compatibility*: Newer code can read data that was written by older code.

*Forward compatibility*: Older code can read data that was written by newer code.

### Formats for Encoding Data

Data representation:

1. In memory: objects, structs (access by CPU by pointer)
2. Send data: self-contained sequence of bytes (no pointer)

1 to 2: *encoding ( serialization or marshalling),*

2 to 1: *decoding  (parsing, deserialization, unmarshalling)*

#### Language-Specific Formats

 Java: java.io.Serializable. Third-party: Kryo
Ruby: Marshal
Python: pickle

Pros: 

- convenient

Problems:

- tied to programming language. 
  You can't change language
  Different to Integrate with others that use different language
- Security problem: decoding will instantiate arbitrary classes
  Hacker can make it executes malicious code
- Versioning
  They are for quick and easy encoding. Neglect forward and backward compatibility
- Efficiency 
  (CPU time, size of the encoded structure)
  Java's built-in serialization is notorious for its bad performance and bloated encoding

#### JSON, XML, and Binary Variants

JSON, XML, and CSV are textual formats, somewhat human-readable

Problems:

- Numbers

  - xml, csv: can not distinguish number and string of number

  - JSON: can not distinguish int and float

  - Large number: 

    int >2^53 can not be represent by double. Thus decoding brings inaccuracy
    Twitter: include Twitter ID in JSON twice, one as number and decimal string

- Doesn't support binary string
  Get around: Base64-encoded
  Hacky and increases data size by 33%
- Schema support
  Correct interpretation depends on schema
  Application need to hardcode it
- CSV
  Value can have comma, newline character

JSON, XML, and CSV

- Good enough for many purposes
- As long as people agree on the format
- Efficiency doesn't matter that much. 
  Getting different organizations to agree on *anything* outweighs most other concerns

**Binary encoding**

TODO

#### Thrift and Protocol Buffers

#### Avro

#### The Merits of Schemas





### Modes of Dataflow

### Summary