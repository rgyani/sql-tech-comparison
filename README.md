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
| 

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

#### B+ trees

B+ trees organize data in **a sorted order**, which makes them ideal for indexing. Each node of the tree contains a range of keys and pointers to child nodes or actual data entries. This sorted structure enables fast search operations, allowing the database to quickly locate the desired data based on the indexed column values.

B+ trees are **balanced trees**, meaning that the height of the tree remains relatively small regardless of the number of entries in the index. This property ensures that search operations have a consistent time complexity, typically O(log n), where n is the number of entries in the index. This balanced nature makes B+ trees efficient for managing large datasets in SQL databases.

#### Page
In a database, a "page" refers to a unit of storage typically used for managing data on disk. Pages are fixed-size blocks of data, often a few kilobytes in size, and they serve as the fundamental unit for reading from and writing to disk.

Now, let's discuss how pages relate to B+Trees and indexes:

1. **B+Tree Structure**: B+Trees are hierarchical data structures composed of nodes. Each node of the B+Tree corresponds to a page in the database storage. These nodes hold keys and pointers that help navigate through the tree structure to locate specific data entries.

2. **Page Organization**: In a B+Tree, the database pages are organized hierarchically, with the root node typically residing in one page, and subsequent levels of the tree stored in additional pages. Leaf nodes, which contain the actual data entries, also correspond to database pages.

3. **Disk I/O Optimization**: Because pages are the units of data read from and written to disk, B+Trees are designed to optimize disk I/O operations. When traversing the tree to search for a specific data entry, the database engine reads only the necessary pages from disk, minimizing the number of I/O operations required. This efficient use of pages helps reduce disk access times and improves overall query performance.

4. **Index Entries**: Database indexes, such as those built using B+Trees, contain entries that map keys to data locations. These index entries are stored within the nodes of the B+Tree, which in turn correspond to database pages. By organizing index entries in a structured manner within the B+Tree nodes, databases can quickly locate the desired data by navigating through the tree structure.

In summary, pages in a database represent units of storage used for managing data on disk. These database pages correspond to the nodes of the B+Tree structure, where index entries contain the column(s) on the table the index is created on.

Remember this is just a bird's-eye view.  
Both MySQL and PostgreSQL support various types of indexes, including B-tree, hash indexes, and full-text indexes. However, PostgreSQL offers more advanced indexing options, such as GiST (Generalized Search Tree) and GIN (Generalized Inverted Index) indexes, which are useful for indexing complex data types like geometric data, text search, and arrays.


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



# TimescaleDB

As we saw earlier, Postgres has great support for extensions.  
TimescaleDB scales PostgreSQL for time-series data via automatic partitioning across time and space (partitioning key), yet retains the standard PostgreSQL interface.

In other words, TimescaleDB exposes what look like regular tables, but are actually only an abstraction (or a virtual view) of many individual tables comprising the actual data. This single-table view, which they call a **hypertable**, is comprised of many chunks, which are created by partitioning the hypertable's data in either one or two dimensions: by a time interval, and by an (optional) "partition key" such as device id, location, user id, etc.