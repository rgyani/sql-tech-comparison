# SQL Tech Comparison

Been working with NoSQL for quite some time now, might be good idea to document the difference in implementation of the SQL favorites MySQL and Postgres for historical purpose.

First a Quick Summary of the differences

| MySQL | PostgreSQL |
|---|---|
| Pure RDBMS | actually an **Object-RDBMS**, which means you can integrate your own types and methods |
| supports numeric, character, date, time, spatial and JSON data types| all MySQL data types along with network addresses, arrays, ranges, XML etc|
| **limited** support for views, triggers, procedures| supports MATERIALIZED VIEWS instead of triggers |
| ACID compliant only with **InnoDB and NDB Cluster storage engines**| Always ACID compliant |
| has B-tree and R-tree index | Supports multiple index types like expression indexes, partial indexes, and hash indexes along with trees.|
| does not offer MVCC | supports MVCC, which creates **duplicate copies of records to safely read and update the same data in parallel** |


As is obvious from the table above, Postgres is more suited for enterprise-grade application with frequent write operations and complex queries, while MySQL has slightly better performance in frequent read scenarios.

> But, as we know there is no silver bullet, consider [Uber's blog post](https://www.uber.com/en-DE/blog/postgres-to-mysql-migration/) where they explain their reason for switching from Postgres to MySQL


### Write performance
MySQL uses write locks to achieve real concurrency. For example, if one user is editing the table, another user may have to wait until the operation finishes before changing the table.

However, PostgreSQL has built-in multiversion concurrency control (MVCC) support without read-write locks. This way, PostgreSQL databases perform better in the case of frequent and concurrent write operations.

### Read performance
**PostgreSQL creates a new system process with significant memory allocation (about 10 MB) for every user connected to the database.** It requires memory-intensive resources to scale for multiple users.

On the other hand, MySQL uses a single process for multiple users. As a result, MySQL database outperforms PostgreSQL for applications that mainly read and display data to users.

# Storage
Let's explore from a bird's-eye view the storage mechanisms of the two databases.  
But first we need to revise Indexes, B+Trees, and Page in DB


### Indexes

In SQL databases, an index is like a table of contents in a book – it helps the database quickly locate specific rows of data based on the values of certain columns. B+ trees are particularly well-suited for this task because they provide efficient searching, insertion, and deletion operations.

### B+ Trees: The Engine of the Index
While many index types exist, the B+ Tree is the golden standard for relational storage engines because it optimizes disk I/O.

- **Balanced Height O(log n):** B+ Trees are perfectly balanced, **meaning every leaf node is exactly the same distance from the root.** No matter how large your dataset grows, looking up a key requires the exact same number of page reads.
- **Routing vs. Storage:**
  * **Internal (Non-Leaf) Nodes** contain only look-ahead keys and pointers to child nodes. They act purely as a roadmap.
  * **Leaf Nodes:** Contain the actual indexed keys and their corresponding data values (either the raw row or a pointer to it).
- **Sequential Leaf Linage:** All leaf nodes are linked together in a continuous chain **(typically a doubly-linked list)**. This allows the database to perform incredibly fast range scans; once the engine finds the starting key, it can simply walk horizontally across the leaves instead of traversing back up and down the tree.

#### Pages: The atomic unit of data storage 
A Page (or Block) is a fixed-size chunk of memory (typically **8KB in Postgres** and **16KB in MySQL's InnoDB**) that serves as the fundamental unit of data transfer between disk and RAM. **Databases never read a single row from disk; they read the entire page containing that row.**

### Step-by-Step: Looking up a Value (e.g., ID = 42)
1. Step 1: Read the Root Page (The top of the tree).
   * The database engine looks at the index and loads the very first page, known as the Root Page (this page is almost always already sitting in RAM cache).
   * Inside this page is a sorted list of keys and page numbers.
   * The engine scans this page to see where 42 falls. For example: "Keys 1 to 50 are on Page#204; keys 51 to 100 are on Page#205."
   * The engine now knows it needs to look at Page#204.
2. Step 2: Read the child/leaf page
   * The engine fetches Page#204 from RAM (or pulls it from the disk if it’s not cached).
   * Because this is a B+Tree, Page#204 is a Leaf Page.
   * The engine looks inside Page#204, scans the data array inside it, finds the key 42, and immediately grabs the corresponding data (the full row in MySQL, or the ctid pointer in Postgres).
   * **Binary Search Inside the Page:** Once a page is loaded into memory, searching inside that specific page is incredibly fast because the keys inside that page are strictly ordered. The database uses binary search to find the exact slot in microseconds.

### Step-by-Step: Writing a new Row Value (e.g., ID = 42)
1. Step 1. Traverse the Tree: 
   * Just like the lookup, the engine reads the Root Page, sees that 43 belongs on Page#204, and loads Page#204 into RAM.
2. Step 2: What happens if the page is already full? Because pages have a fixed size (e.g., 8KB or 16KB), a database can't just keep stuffing data into them forever.
   * Shift and Insert: The engine sees there is still empty space on Page#204. 
      - Because keys must stay strictly sorted, it shifts 44 and 45 over slightly, drops 43 into its correct slot, and writes the page back to disk/cache.
   * The "Page Split": The engine sees the page is full
      - The Database Allocates a Brand New Page (#206) and splits the data 50/50:
         ```
         [ Page #204 ] ──► [ 40 | 41 | 42 ]
         [ Page #206 ] ──► [ 43 | 44 | 45 ]  <-- 43 safely inserted here
         ```
      - The Parent/Root Page is updated to point to both pages:
         ```
         [ Root Page ] ──► Points to Page#204 (for keys <= 42) 
                       ──► Points to Page #206 (for keys >= 43)
         ```

### This is reason why we prefer auto-increment IDs vs UUId for primary keys
* With auto-increment Ids, New inserts always go at the very end of the last page. Pages fill up sequentially, and when a page splits, it only ever splits at the very edge.
* If you use random UUIDs, a new insert could target a page right in the middle of your tree. If that page is full, it forces a random page split, shattering performance and leaving pages only 50% full (causing fragmentation and wasting space).


### How Postgres and MySQL differ
#### 1. Postgres: The Two-Pile System
Postgres keeps its Index boxes completely separate from its Data boxes.
* **The Heap Pages (Data Pile):** You have a giant pile of boxes that contain only raw rows of data. They are in no particular order. This is called the Heap.
* **The Index Pages (B+Tree Pile):** You have a separate pile of boxes arranged as a B+Tree.
* **How they connect:** You open an Index box, find your key, and it gives you a ctid (a slip of paper). That slip of paper says: "Go over to the Heap pile, open Page Box #500, and look at row Slot #3."

```
[ Index Page Box ] ──► Contains: "ID 42 is in Data Box #500, Slot 3"
                            │
                            ▼
[ Heap Data Page Box #500 ] ──► Contains: [ Actual Row Data for ID 42 ]
```
#### 2. MySQL: The All-in-One System
MySQL (InnoDB) decides that having two separate piles of boxes is a waste of time for the primary key. It merges them.
* **The Clustered Index Pages:** MySQL takes the raw data rows and stuffs them directly inside the leaf boxes of the B+Tree itself.
* **How it works:** You traverse the B+Tree boxes. When you get to the final leaf Page Box, you don't find a note telling you to go look at another box. You open that page box, and the actual data row is sitting right inside it.
```
[ Primary Key Leaf Page Box ] ──► Contains: [ Actual Row Data for ID 42 ] 
                                  (No next step needed!)
```

#### To Put It Simply:
* Does Postgres use pages? Yes. It reads an Index Page, gets a pointer, and then reads a Heap Data Page.
* Does MySQL use pages? Yes. But for the primary key, the Index Page is the Data Page. Finding the right page means you have found the data.

Remember this is just a bird's-eye view.  
Both MySQL and PostgreSQL support various types of indexes, including B-tree, hash indexes, and full-text indexes. However, PostgreSQL offers more advanced indexing options, such as GiST (Generalized Search Tree) and GIN (Generalized Inverted Index) indexes, which are useful for indexing complex data types like geometric data, text search, and arrays.


## Secondary Indexes
### MySQL
In a MySQL database, we *must* have a primary index, whose value is the full row.  
If you do a lookup *for a key in the primary index* you find the page where the key lives (O(logN) operation) and its value which is the full row of that key, no more I/Os are necessary to get additional columns.

In a secondary index the key is whatever column(s) you indexed and the value is a pointer to where the full row really lives. **The value of secondary index leaf pages are usually primary keys.** So when we do a lookup for a key in secondary index, we first find the value of the primary key (O(logN) operation) and then another lookup to get the actual value (another O(logN) operation)


### Postgres
In PostgreSQL, there is no concept of user defined primary key, but there is a system managed **immutable tuple** called ctid. A ctid conceptually represents the on-disk location (i.e., physical disk offset) for a tuple.  
Now each index, (primary key(s) and other secondary indexes) maps the columns to the ctid and is stored as a B-tree.
So now a look for primary key, as well as secondary key, will fetch the ctid (O(logN) operation) and then the row data can be fetched using a O(1)

This looks better already since, instead of two O(LogN) operations, we just have a single O(logN) operation.  
However...

### Updates
In MySQL updating a column that is not indexed will result in only updating the leaf page where the row is with the new value. No other secondary indexes need to be updated because all of them point to the primary key which didn’t change.

In Postgres updating a column that is not indexed will generate a new **immutable tuple** (the core idea behind multi-version concurrency control) and might require ALL secondary indexes to be updated with the new tuple id because they only know about the old tuple id. This can potentially causes many write I/Os and needs more storage space. Uber didn’t like this one in particular back in 2016, one of their main reasons to switch to MySQL from Postgres.  

However...  
since Postgres 8.3, they have implemented a feature called Heap Only Tuples (HOT), in which the updated row is stored on the same table page as the old row, with 
* a HEAP_HOT_UPDATED bit set on old row to indicate this tuple has been updated
* a HEAP_ONLY_TUPLE bit set on new row to indicate this tuple does not have an index entry of its own

 It's important to note that HOT updates have limitations. They only apply when the updated row can remain on the same page and doesn't affect any indexed columns. If the updated row can't fit on the same page or affects indexed columns, PostgreSQL will perform a regular update, which may create table bloat and require index updates.

# Data types of Primary Key
* In MySQL choosing the primary key data type is critical, as that key will live in all secondary indexes. For example a UUID primary key will bloat all secondary indexes size causing more storage and read I/Os.
* In Postgres the tuple id is fixed 4 bytes so the secondary indexes won’t have the UUID values but just the ctids pointing the heap.


# Processes vs Threads Model
* Historically, MySQL has used a thread-based model, where each connection to the database is associated with a separate thread. These threads handle client connections and execute queries.     
* PostgreSQL, on the other hand, traditionally uses a process-based model, where each connection is handled by a separate operating system process. These processes are managed by the PostgreSQL server and are responsible for executing queries and managing transactional state. However, creating new processes can be more resource-intensive than creating threads, so PostgreSQL typically has a higher memory overhead per connection compared to MySQL. 


## Postgres MVCC 
Postgres's MVCC operates on an append-only model—very similar to how an LSM-Tree (Log-Structured Merge-tree) handles writes in databases like Cassandra or RocksDB.

Here is exactly how Postgres handles high-write workloads, why it causes massive "bloat," and how it contrasts with MySQL.

### 1. The Core Issue: "Pay Later" vs. "Pay Now"
```
PostgreSQL (Append-Only / Pay Later)
[ Page 1 ] ───► Old Tuple (Dead) ───► New Tuple (Active)
* Result: High write throughput, but leaves dead tuples that require VACUUM.

MySQL / InnoDB (In-Place / Pay Now)
[ Page 1 ] ───► Overwrites Tuple In-Place
   │
   ▼ (Stashes changes)
[ Undo Log ] ───► Delta History
* Result: Slower writes due to processing, but zero table bloat.
```

##### PostgreSQL: The "Pay Later" Append-Only Model
When you run an UPDATE in Postgres, it does not overwrite the data. It copies the entire row, applies the change, and appends a brand new row (tuple) onto the disk page. The old row is marked as "dead."

The Good: Writes are incredibly fast because Postgres just dumps data sequentially into memory and writes to the WAL (Write-Ahead Log).  
The Bad: It creates massive Table Bloat. If you update a single row 50,000 times, you physically create 50,000 dead tuples.

##### MySQL (InnoDB): The "Pay Now" In-Place Model
MySQL updates data in-place. It overwrites the row directly on the data page. To preserve multi-version isolation, it copies only the modified columns into a separate Undo Log.

The Good: No table bloat. The table size remains tightly constrained.  
The Bad: Writes are slower because MySQL has to do the structural work up front (stashing to Undo log, writing Redo log, updating the page). Long-running reads can also slow down because they have to reconstruct history by traversing the undo log chain backwards.

### 2. How Postgres Mitigates the High-Write Dilemma
Because Postgres engineers knew that high updates would kill performance, they built two native defense mechanisms.

#### Defense 1: HOT (Heap-Only Tuples) Optimization
Postgres avoids rewriting index pointers on every update through an optimization called HOT.

If you update a row, and:
* The modified column is not indexed.
* The new version of the row can fit inside the same 8KB data page as the old version.
Postgres will not create a new index pointer. Instead, it creates a lightweight pointer chain inside the data page from the old row to the new row.

Furthermore, any subsequent SELECT query that hits that page will silently prune the dead tuple on the spot. This means a lot of garbage collection happens instantly during standard read traffic without needing the global VACUUM process.

#### Defense 2: Aggressive Autovacuum Tuning
For high-write scenarios, the default Postgres autovacuum settings are way too passive. At scale, we aggressively tune the autovacuum daemon to run continuously and lightly, rather than periodically and heavily.

In a high-update environment, you tweak these settings in postgresql.conf:
```
# Scale down thresholds so autovacuum triggers much faster
autovacuum_vacuum_scale_factor = 0.05  # Trigger when 5% of rows are dead (default is 20%)
autovacuum_vacuum_threshold = 500       # Trigger after 500 oboslete rows

# Give vacuum more muscle to finish quickly
autovacuum_max_workers = 6             # Run more parallel worker threads
autovacuum_vacuum_cost_limit = 2000    # Let workers do more I/O before sleeping
```

#### 3. If you face the issue "Even with HOT and tuned Autovacuum, our update traffic is still causing unacceptable table bloat. What do you do?"
 - Shun "In-Place" Relational Updates: If you are updating a status column constantly (e.g., status = 'processing' -> 'completed'), don't update. Use an Append-Only Event Ledger pattern. Treat Postgres like a NoSQL event log. Just write an INSERT for every state change, and query the latest record using a window function or a dedicated materialized view.
- Vertical Partitioning (Split the Table): If you have a table with 50 columns, but only 2 of those columns are updated frequently, split those 2 columns out into a separate 1-to-1 extension table. This keeps the primary, massive text columns from bloating, restricting the MVCC duplication to a tiny, fast-moving secondary table.


# TimescaleDB

As we saw earlier, Postgres has great support for extensions.  
TimescaleDB scales PostgreSQL for time-series data via automatic partitioning across time and space (partitioning key), yet retains the standard PostgreSQL interface.

In other words, TimescaleDB exposes what look like regular tables, but are actually only an abstraction (or a virtual view) of many individual tables comprising the actual data. This single-table view, which they call a **hypertable**, is comprised of many chunks, which are created by partitioning the hypertable's data in either one or two dimensions: by a time interval, and by an (optional) "partition key" such as device id, location, user id, etc.

## "Postgres is eating the database world"
Do read for an opinionated sales pitch for Postgres [https://pigsty.io/blog/pg/pg-eat-db-world/](https://pigsty.io/blog/pg/pg-eat-db-world/)

## 28 Jan 2025,  Microsoft has launched a new open-source document database platform built on PostgreSQL

Microsoft has launched a new open-source document database platform built on PostgreSQL. This platform offers no commercial licensing fees and enhances performance with BSON support. **Developers can use FerretDB 2.0 for a MongoDB-like experience with improved performance.**

### Key Features of Microsoft’s Open Source Document Database
The platform includes two custom PostgreSQL extensions designed to optimize performance and efficiency:

1. **Pg_documentdb_core:** This extension optimises PostgreSQL for BSON (Binary JavaScript Object Notation) handling. BSON is a binary-encoded format of JSON documents, making it much faster to store and retrieve data.
2. **Pg_documentdb_api:** This extension adds the ability to perform CRUD operations (Create, Read, Update, Delete), manage queries, and index your data.


Together, these extensions provide the foundation for building document-based applications on top of a relational system.
