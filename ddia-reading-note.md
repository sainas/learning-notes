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

**JSON Binary encoding**

JSON binary encoding: no prescribe schema, very small space reduction. Not worth loss of human-readability



#### Thrift and Protocol Buffers

Both require a schema

Thrift: *BinaryProtocol* and *CompactProtocol*

type + field tag + length indication + data

CompactProtocol:

- one byte for field type and tag number
- Smaller integer use less bytes

Required and optional: they make no diff on encoding. Only "required" enables a runtime check.

**Schema Evolution**

1. Field tags
   - Free to change field name, but can not change field tag
   - Adding new field:
     - Forward compatibility: meets. Old code can ignore (datatype annotation tells it how many bytes to skip)
     - Backward compatibility: new field can not be required. Must be optional or with a default value
   - Removing fields:
     - Can only remove field that is optional
     - Can never reuse the same tag number
2. Datatypes
   - Risk: lose precision or get truncated.
     e.g. 32-bit int to 64-bit int. Old code will truncate the value
   - *Protocol Buffers* doesn't have a list or array data type.
     It's okay to change optional field(single-valued) to repeated field(multi-valued)
     And old code only sees the last element of the list
   - *Thrift*: can't make the above evolution. But has the advantage of supporting nested lists

#### Avro

Most compact

- Schema: No tag number
- Binary data, no fields or datatypes. Only in schema

Any mismatch in the schema would cause trouble

**The writer’s schema and the reader’s schema**

writer’s schema and reader’s schema don't have to be the same

Can handle: different order, ignore fields, autofill with default value

**Schema evolution rules**

To maintain forward and backward compatibility, only add or remove a field with a default value

Only backward compatible: adding alias for field, add one more union type

**But what is the writer’s schema?**

Usecase:

1. Hadoop: large file with lots of record
2. Database with individually written records
   Version number at the beginning of encoded record, and a list of schema version
3. Sending records over a network connection
   Negotiate the schema version on connection setup

**Dynamically generated schemas**

Advantage of Avro: Friendlier to dynamically generated schemas

No tag numbers. No need manual mapping and avoiding used numbers

**Code generation and dynamically typed languages**

Code generation are more for statically typed languages, not for dynamically typed languages, since they don't have compile-time type checker

For Avro (dynamically generated schema from db table), code generation is unnecessary

Avro container file is self-describing (embeds with writer's schema)

#### The Merits of Schemas

Many data systems also implement binary encoding:
e.g. relational databases have a network protocol for queries/responses
Database vendor provides a driver (e.g., using the ODBC or JDBC APIs)  decodes responses from the database’s network protocol into in-memory data structures.

Pros of using schema:

1. schema language are simplaer than JSON
2. Support detailed validation rules

Pros of binary coding:

1. more compact (omit field names)
2. schema is valuable doc, and guarantee to be up-to-date (unlike manually maintained doc)
3. Keeping a database of schemas allows you to check forward/backward compatibility, before deploy
4. Able to generate code from schema, enables type checking at compile time

### Modes of Dataflow

- Via databases
- Via service calls (REST and RPC)
-  Via asynchronous message passing

**Dataflow Through Databases**

More than backward and forward compatibility:

New code writes in, with a new field, old code(doesn't know about new field) reads and updates this record. The new field can be lost.
Desirable behavior is: old code keeps the new field intact, even though it couldn’t be interpreted.

**Different values written at different times**

Same DB have five-year-old data and 5-min-old data

Unlike code deployment replaces all the code

***"Data outlives code"***

Migrating is expensive

Simpler schema change: New column with null default value

Schema evolution: the entire database looks like it was encoded with a single schema, which actually was by various historical versions of the schema.

**Archival storage**

data warehouse

Encoded by latest schema (you are copying the data anyway)

Encode the data in an analytics-friendly column-oriented format such as Parquet

#### Dataflow Through Services: REST and RPC

*clients* and *servers*

The API exposed by the server is known as a *service*

Server can itself be a client to another service: *microservices architecture*

Services are in some way like databases, but the API are predetermined

**Web services**

HTTP is used as protocol

Not only used on the web

- mobile app (native app, or JS web app using Ajax)
- One service to another service within same organization (*middleware*)
- One service to another service owned by different organization, via internet (public API, OAuth)

REST: not a protocol but a design philosophy 
An API designed according to the principles of REST is called RESTful.

SOAP: use an XML-based language (WSDL)

SOAP's Cons:

- WSDL not human readable
- Too complex and rely on tool support. Difficult to integrate if programming language not supported by SOAP vendor
  Interoperability is not good

RESTful APIs' Pros:

- simpler, less code generation
- A definition format Swagger, to describe RESTful APIs

**RPC (remote procedure calls): Problems**

Make a request just as calling a function "location transparency"

Fundamentally flawed, because

- A local function call is predictable: succeeds or fails
  A network request is unpredictable: lost due to network problem, slowness

  You need to anticipate them, e.g. by retrying

- A local function call return result or throws exception (or infinite loop)
  A network request: return without result, not return due to a timeout
  In this case you don't know what happened to the service

- If network request fails, it could be 

  - Request failed
  - Request got through, just responses are lost. 

  Need to build some mechanism to make the retries *idempotence*

- Local function takes about same time, network latency is unpredictable (network, load of the remote service)

- Local function can use references (pointers). Network must encode/decode

- Different programming language, RPC need to translate datatypes
  Ugly because not all languages have the same type. e.g. a number > 2^53 in JS

**RPC (remote procedure calls): Current directions**

It isn't going away. Many frameworks built on it.

New generation: distinguish remote request from local function
*futures (promises)* to encapsulate asynchronous actions that may fail
*service discovery*: finds the IP & port of a particular service

Customer RPC with binary encoding has better performance
But JSON is easier to debug (e.g. command-line tool `curl`, and a vast ecosystem of tools)

REST is the predominant style. RPC mainly focus is between services within same organization, typically same data center

**RPC (remote procedure calls): Data encoding and evolution**

Servers update first, clients second, thus

- Backward compatibility on requests
- Forward compatibility on response

Hard since can not force client to upgrade

- Need maintain compatibility for a long time
- If breaking change needed, ends up maintaining multiple versions of API

API Versioning:
RESTful: version in URL or HTTP header
Or if client uses API key, store the version in DB and modified through a separate admin interface

#### Message-Passing Dataflow

*asynchronous message-passing*

|       | REST & RPC  | Database                     | Message-Passing                                              |
| ----- | ----------- | ---------------------------- | ------------------------------------------------------------ |
| Speed | Immediately | Store and read in the future | In between, like both: client's request is delivered with low latency. Message not sent via direct network connection, but with an intermediary |

*message broker* (*message queue* or message-oriented middleware): stores the message temporarily.

Message broker Pros: (over RPC)

- Act as a buffer (improve reliability)
- Auto redeliver
- Avoid sender needing to know IP & port of recipient (Useful in cloud, virtual machines come and go)
- One message to several receipents
- Logically decouple sender from recipient (sender only public message, doesn't care who consumes)

Different from RPC: only one-way communication

**Message brokers**

producers   --publish-->  *queue* or *topic*   <--subscribe--  *consumers* or *subscribers*

Many to many

One topic is one-way, but a consumer can publish too, to a reply queue

Typically any encoding format works

If a consumer republish message, need to preserve unknown field (like the same issue in DB)

**Distributed actor frameworks**

*actor model*:

- Each actor has local state(not shared), communicates asynchronously

- Concurrency in a single process (no threads, thus no race conditions, locking, deadlock)

- *Location transparency* is better than RPC since actors assumes messages might be lost

*distributed actor framework*: actor + message broker
Still need to worry about compatibility

### Summary

Rolling upgrade: allows no downtime, less risky (roll back before affecting more user)

These properties are hugely beneficial for "*evolvability*"

Backward and forward compatibility
(Must assume different versions are running)

| Data encoding format                                         | Compatibility                                                |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| Programming language–specific encodings                      | Often failed to provide                                      |
| Textual: JSON, XML, CSV                                      | Depends. Vague on datatype thus need to be careful with numbers and binary strings |
| Binary schema-driven format: Thrift, Protocol Buffers and Avro | Compact. Clear defined compatibility semantics. Schema can be useful doc. Code generation in statically typed language. Not human-readable |

| Dataflow                     | Case                                                         |
| ---------------------------- | ------------------------------------------------------------ |
| Database                     | Writer encodes, reader decodes                               |
| RPC and RESTful API          | Client encodes requests, server decodes, encodes the response. Client decodes response. |
| Asynchronous message passing | Sender encodes, recipient decodes                            |

# PART II. Distributed Data

Why multiple machine?

1. Scalability
2. Availability
3. Latency (geo location)

Scaling to Higher Load:

1. Vertical scaling/Scaling up
   - Shared-memory architecture
     Problem: price is not linear, and efficiency gain is worst than linear
     Not fault tolerance. (But some allows hot-swappable)
     One geo location
   - Shared-disk architecture
     Used for data warehousing workloads
     Locking is difficult to handle
2. Horizontal scaling/Scaling out
   - Shared-nothing architecture
     With cloud virtual machines now multi-region is feasible for even small companies

Ways to distribute data across nodes:

- Replication
- Partition

## Chapter 5. Replication

Why replicate?

- Geographically closer to user(Lower latency)
- failover(high availability)
- more nodes to serve read (high read throughput)

Three popular algorithms

- single-leader
- multi-leader
- leaderless replication

### Leaders and Followers

1. Clients must send write request to leader. Leader writes to local storage
2. Leader sends data to followers as *replication log* or *change stream*
   Followers apply the writes in the same order
3. Read can be handle by both leader and followers. Writes are only accepted on the leader

Built-in feature of many db e.g. PostgreSQL, MySQL, MongoDB
And some message queue Kafka RabbitMQ highly availability queues

#### Synchronous Versus Asynchronous Replication

When the leader notify the client that the update has been successful?
Wait until the follower confirms the write: That follower is synchronous
Not wait for the follower: That follower is asynchronous

| --   | Synchronous                                                  | Asynchronous    |
| ---- | ------------------------------------------------------------ | --------------- |
| Pros | Guarantee consistency                                        | Not up-to-date  |
| Cons | Leader blocks write if one follower doesn't respond (crashed or network issue) | Not block write |

Impractical if all followers are synchronous

*semi-synchronous:*

- usually one synchronous and others async 
  If that one sync unavailable or slow, another async is made sync

- This make sure two up-to-date copies

*completely asynchronous*

- Write can be lost if leader failed (Weakening durability)
- Can continue processing write even all followers fall behind

Research on not losing data if leader fails:
*chain replication*

#### Setting Up New Followers

Just copy is not sufficient, since data is always in flux
Could lock the DB: against goal of high availability

Process

1. Snapshot leader's DB (without lock the entire db)
2. copy snapshot to new node
3. new node connects to leader and request data changes in between (snapshot is associated with one position in leader's replication log)
4. New node process the backlog and *"caught up"*

#### Handling Node Outages

e.g. Reboot to install security patch. Reboot without downtime

**Follower failure: Catch-up recovery**

Follower has all data change logs received from leader

Request to leader for changes since the failure

**Leader failure: Failover**

Promote one follower

- Client need to send writes to new leader
- Other followers need to consume data from new leader

This is called *failover*

Steps

1. Determining that the leader has failed.
   Nodes bounce messages between each other, if no response in long time
2. Choosing a new leader
   Election process, or appointed by *controller node*. Consensus problem
   Best candidate: most up-to-date
3. Reconfiguring the system
   If the old leader comes back, system need to handle it

Failover is fraught with things that can go wrong:

- asynchronous replication: data loss
  If old leader rejoin: what to do with those writes?
  Common solution: discard. May violate durability expectation

- Discarding writes can be dangerous if other storage system coordinates with DB
  GitHub autoincrement primary key. New leader lagged, reuse some primary keys, also used in Redis, caused private data to be disclosed to the wrong users
- Two new leaders: *split brain*
  data lost/corrupted. Two node accepts conflict writes
  Solution: mechanism to shut one down. But if not careful can shut two down
- What's the right time out for leader to be declared dead?
  Too long: long recovery
  Too short: unnecessary, load spike/network issue, making things worse

No easy solution. Some team prefers to do failovers manual

#### Implementation of Replication Logs

##### 1. Statement-based replication

SQL statement

Cons:

- nondeterministic functions: NOW()  RAND()
- autoincrementing column, or depends on the data (UPDATE...WHERE...)
  Must be same order, but there could be concurrent transactions
- side effects: triggers, user-defined functions

Workaround: replace nondeterministic with fixed return
But still many edge cases

##### 2. Write-ahead log (WAL) shipping

Main disadvantage: log are very low level (which bytes were changed.)
If database storage format version changes, not possible
Follower and leader must be same version

Thus can't do zero-downtime upgrade

##### 3. Logical (row-based) log replication

Different log format for storage and replication  (Decoupling)

logical log (compared to physical data)

- Insert: new value
- Delete: info to uniquely define the row. PK if the table have one, otherwise values
- Update: uniquely defined row, and new values of (all or changed) column

Pros:

- Can be kept backward compatible within different version of software or even storage engine
- Easier for external app to parse (e.g. data warehouse, building custom indexes and cache)

##### 4. Trigger-based replication

1-3 are done by database system. No application code involved

More overhead, but more flexible

Many db have this functionality
*triggers and stored procedures*

A trigger lets you register custom application code that is automatically executed when a data change (write transaction) 



### Problems with Replication Lag

Leader-based replication:  Attractive for read heavy system

*read-scaling* architecture: many followers (BUT must be asynchronous)

Asyn follower: temp inconsistency

When the followeres catch up: eventual consistency

#### Reading Your own Writes

*read-after-write consistency*   OR *read-your-writes consistency*

How to implement

- **(All leader)**Things user may modify -> read only from leader
  How to know what are the things user may modify?
  e.g. user profile
  user's own profile -> always from leader
  other user's profile -> always from followers
- **(Earlier Leader, then followers)**If most things can be modified by user, that is not effictive
  Track time of last update
  1min from last update: read from leader
  1min+ from last update: choose followers with lag < 1min
- **(Followers: choose by lag)**Client can remember timestamp of most recent write
  Choose replica with last update >= that timestamp
  (If not sufficiently up to date: choose another, or wait)

More complex with multi-datacenter:
Request to leader must be routed to the datacenter that leader is in

More complex with use accessing from multi-device
*cross-device* read-after-write consistency:

- Remembering last update timestamp is more difficult
  Metadata will need to be centralized
- Requests from different devices may be routed to different datacenter
  Need to make sure route to the same datacenter first if must read from the leader

#### Monotonic Reads

*moving backward in time*

If you query a follower with lag A, then another follower with lag B, B > A

*strong consistency > Monotonic reads > eventual consistency*

How to achieve:
User always read from same replica
e.g. replica chosed by hask of userID

Problem:
When replica fails, will need rerounting

#### Consistent Prefix Reads

concerns violation of causality

If some partitions are replicated slower than others, an observer may see the answer before they see the question.

*Consistent Prefix Reads*

Guareetee if writes happen in an order, reading those will see the same order

Particular problem in partitioned database, many have no global ordering writes

Solution:

1. Any causally related writes to the same partition (sometimes not efficienly)
2. Algorithm "happen-before"  page 186

#### Solutions for Replication Lag

Those stronger guarantee, such as read-after-write, pretends the replication is synchronous when it's actually asynchronous for minutes

Application can provide stronger guarantee than DB: e.g. run certain reads only on leader (more complexity)

But it's better developer didn't have to worry about repliaction issue

That's why *transaction* exist

Single-node transaction: simple
Distributed db: many have abandoned it, too expensive

### Multi-Leader Replication

One leader downside: all write must go through it
If can't connet to the one leader, can't write to DB



#### Use Cases for Multi-Leader Replication

##### 1. Multi-datacenter operation

Good for:

- Performance
  Less latency to writes

- Tolerance of datacenter outages

  If a datacenter fails:
  Single leader: need to promote a follower of another data center
  Multi-leader: can continue operating till it's back

- Tolenrance of network problem
  Traffic between datacenter uses public internet (less reliable than local network)
  Single-leader: sensitive to inter-datacenter link
  Multi-leader: better. Network interruption doesn't block writes

Downside:
Conflicts

Dangerous (autoinrement, triggers, integrity constrains can be troublesome)

##### 2. Clients with offline operation

Like calendar app on your phone, every device's local db acts as a leader

Replication lag can be days

##### 3. Collaborative editing

*Real-time collaborative editing* 

e.g.Google Doc

Not a database replication problem, but similar

Need lock on the doc, but unit must be small (single key stroke)

#### Handling Write Conflicts

e.g.  
User 1 changes title from A -> B
Uder 2 changes titile from A -> C
Both consider successful to local leader

Approach

1. **Conflict avoidance**
   Since many implementations of multi-leader replication handleconflicts quite poorly, avoiding conflicts is a frequently recommended approach
   e.g. user can edit profile data. Make these requests always route to the same datacenter "home" datacenter.
   From user's point of view it's like single-leader
   (But designated leader may change due to datacenter failure or user moving)
2. **Converging toward a consistent state**
   Can't apply the last writes:
   In the example, leader 1 sees B then C. Leader 2 sees C then B
   Must eventually be the same
   the database must resolve the conflict in a *convergent* way
   - each write a unique ID, highest ID wins
     timestamp: *last write wins*
     Or random number, hash...
     Can cause data loss
   - Each replica a unique ID, writes on higher-numbered replica wins
     Can cause data loss
   - Merge values. Sort alphabetically then concatenate
     i.e.  B/C
   - Preserve all info, write application code that resolves the conflicts later 
     e.g. prompting user
3. **Custom conflict resolution logic**
   Resolution logic in application code
   - On writes
     Can not prompt user
   - On read
     Save all versions and may prompt user or automatically resolve

**Automatic conflict Resolution**

- *Conflict-free replicated datatypes* (CRDTs) 
  e.g. sets, maps, ordered lists, counters, etc
  automatically resolve conflicts
- *Mergeable persistent data structures*
  like Git, and use three-way merge function
  CRDTs use two-way merges
- *Operational transformation* 
  Conflict resolution algorithm behind collaborative editing applications such as Google Docs 
  Concurrent editing of an ordered list of items (such as the list of characters that constitute a text document)

**What is a conflict?**

Some inobvious conflicts:

Two different people booked the same room at the same time.

The application need to ensure one room booked by one group at one time. Even application checks availability before writes, it can still happen on two leaders.

#### Multi-Leader Replication Topologies

In what senquence leader sends to other leader

- All-to-all topology
- Star topology 
  one root node forwards writes
- Circular topology
  All node need to forward writes
  Need to prevent infinite loop

Problem with Star/circular: single point of failure
More densely connected topology is better since there are other paths

Problem with all-to-all: writes arrives in wrong order
(e.g. UPDATE arrives before INSERT)

Similar to "Consistent Prefix Reads"

Solution:

(NOT WORK) attach timestamp. since clocks CANNOT be trusted in sync

1. *version vectors*

But conflict detectioin are poorly implemented in many multi-leader replication system

### Leaderless Replication

Dynamo-Style

Cassandra, Riak, and Voldemort

（DynamoDB != Dynamo, DynamoDB is still single-leader)

Client sends to several replicas, or sent to coordinator node(doesn't enforce ordering of write)

#### Writing to the Database When a Node Is Down

If leader failed: need failover
But in leaderless system, failover doesn't exist

Quorum write, quorum read, and read repair after a node outage

Deal with stale values: Version number

##### Read repair and anti-entropy

How unavaibale node catches up when it's back?

1. Read repair
   Works well for values with frequent read
2. Anti-entropy process
   Background process
   Different from "replication log" in leader-based replication: this doesn't copy write in particular order. Delay can be huge

##### Quorums for reading and writing

w + r > n

Reads and writes that obey these r and w values are called *quorum* reads and writes

*strict quorum*, to contrast with *sloppy quorums*

There maybe more than n nodes in the cluster, but any given value is stored only on n nodes (partitioned dataset)

Still sent request to all n nodes. Just only wait for partial to report success

#### Limitations of Quorum Consistency

w + r ≤ n

More likely to read stale value
Higher availability

w + r > n can hit stale value too, e.g.

- Sloppy quorum
- Two concurrent writes
- Write concurrent with read (not sure new or old value)
- Succeeded only on partial of w, thuse the write is considered failed. But the read may get the value
- Node with new value fails, restored from an old replica
- Timing “Linearizability and quorums” on page 334

Not easy to guarantee.
Dynamo-style is optimized for use case that can tolerent eventual consistency

w and r adjust the probability of stale value being read

No guarantee of *reading your writes, monotonic reads, or consistent prefixreads*

Stronger guarentee would require transactions or consensus

**Monitoring staleness**

Even you can tolerate stale read, you need monitoring
Huge lag: network or overloaded node

In leader-based: writes apply in same order
Can measure lag by subtracing positions

In leaderless: difficult

Eventual consistency is vague, but still we need to be able to measure it
*staleness measurements*

#### Sloppy Quorums and Hinted Handoff

Network can easily cut off a client from a large number of db

Trade-off:

- Is it better to return error to all requests (for not able to reach quorum)?
- Or should we accept the writes and write to nodes not in the n nodes

Later knowns as *sloppy quorum*

Still w and r, but not necessarily the n "home" node

Once network is fixed, those writes are sent to the appropriate "home"
*hinted handoff*

A sloppy quorum actually isn’t a quorum at all in the traditional sense.
No guarantee read from r nodes will see new value until the hinted handoff has completed.

**Multi-datacenter operation**

Multi-leader, leaderless are both suitable for multi-datacenter operation
Tolerate: conflicting concurrent writes, network issue, latency spikes

*Cassandra and Voldemort*: n in all datacenters, but only wait for quorum in local datacenter

*Riak*: n in local data center. Async cross datacenter

#### Detecting Concurrent Writes

Similar to multi-leader write conflicts, but can happen also in read repair and hinted handoff

**Last write wins** (discarding concurrent writes)

Force an arbitraty order: attach a timestamp

LWW achieves the goal of *eventual convergence*, but *at the cost of durability*

Many writes "succeeded", only one survive, other silently discard

Even drop writes that are not concurrent

LLW is the only supported conflict resolution in Cassandra

When to use LWW:
when lost data is acceptable

How to safely use Cassandra:
ensure a key is only writen once (e.g. use UUID as key)

**The “happens-before” relationship and concurrency**

How to decide whether two operatio concrrent or not?

- UPDATE on a INSERT value: casually dependent
- No causal dependencies

Operation A *happens before* another operation B if B knows about A, or depends on A

Concurrent: neither *happens before* the other

We need an algorithm to tell whether concurrent

Concurrency may seems occur "at the same time". But acutally time doesn't matter, since distributed system, hard to tell whether they happened at the same time.

Physical: (special theory of relativity) in physicsinformation cannot travel faster than speed of light

Computer science:

- Two events happen in short time with long distance: can't affect each other
- Two events happen apart for a long time can still be concurrent, due to network slowness

**Capturing the happens-before relationship**

Two people adding things to shopping cart:

The clients are never fully up to date with the data on the server, since there is always another operation going on concurrently.
But old versions of the value do get overwritten eventually, and no writes are lost.

Algorithm:

- Server maintains a version number for every key
  Store value with version number
- Read: get all values haven't been overwritten, and latest version number.
  Client must read key before write
- Write: must include the version number from prior read, and must merge values
- Server receives write with a version number: can overwrite values <= that version. Keep value with higher version

**Merging concurrently written values**

No data loss

Extra cleanup work: merging *siblings*

Simple case: shopping cart: just take the union

*Remove things*: deletion marker *tombstone*

Since merging is complex and error-prone, CRDT is desgined to auto merge in a sensible way

**Version vectors**

Previous example: single replica

How about multi-replica: single version -> version per replica

version vector



Version vectors != vector clocks
Version vectors: track data in replicas
Vector clocks: track events in processes

### Summary

Replication purpose

- High availability
- Disconnected operation
- Latency (geographic, localize data)
- Scalability

Three main approaches

- Single-leader
  Easy, no conflict
- Multi-leader
- Leaderless

Multi-leader/leaderless: More robust, at the cost of harder to reason about, weak consistency guarantee

Synchronous vs Asynchronous replication
Async: if leader fails, promoting async followers case data lost

Consistencyt models (under replication lag):

- Read-after-write consistency
  See data write by themself
- Monotonic read
  Don't see data older than last seen
- Consistent prefix reads
  Data make causal sense: e.g. not see answer before question

Concurrency and conflict
Whether concurrent is not about time
Resolve conflict by merging

