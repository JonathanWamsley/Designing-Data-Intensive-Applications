# Storage and Retrieval

On the most fundamental level, a database needs to do two things: when you give it some data, it should store the data, and when you ask it again later, it should give the data back to you.  

In Chapter 2 we discussed data models and query languages - i.e., the format in which you (the application developer) give the database your data, and the mechanism by which you can ask for it again later. In this chapter, we discuess the same from the database's point of view: how we can store the data that we are given, and how we can find it again when we are asked for it.  

Why should you, as an application developer, care how the database handles storage and retrieval internally? you are probably not going to implement your own storage engine from sratch, but you do need to select a storage engine that is appropriate for your application, from the many that are available. In order to tune a storage engine to perform well on your kind of workload, you need to have a rough idea of what the storage engine is doing under the hood.  

In particular, there is a big difference between storage engine that are optimized for transactional workloads and those that are optimized for analytics. We will explore that distinction later in 'Transaction Processing or Analtics', and in 'column oriented storage'. we will discuess a family of storage engines that is optimized for analytics.  

However, first we will start this chapter by talking about storage engines that are used in the kinds of databases that you are probably familiar with: traditional relational databases, and NoSQL databases. We will examine wo families of storage engines: log-structured storage engines and page-oriented storage engines such as B-trees  

### Data Structures that power your database

Consider the world's simplest database, implemented as two bash functions

<p>
    #!/bin/bash
    
    db_set() {
        eacho "$1,$2" >> database
    }
    
    db_get() {
        grep "^$1," database | sed -e "s/^$1,//" | tail -n 1
    }
</p>

These two functions implement a key-value store. You can call db_set key value, which will store key and value in the database. The key and value can be (almost) anything you like - for example, the value could be a JSON document. You can then call db_get key whchi looks up the most recent value assocaited with the particular key and returns it.  

And it works:

<p>
    db_set 42 '{"name":"San Franciso", "attractions":["Golden Gate Bridge"]}'
    db_get 42
        {"name":"San Franciso", "attractions":["Golden Gate Bridge"]}
</p>

The underlying storage format is very simple: a text file where each line contains a key-value pair, separated by a comma. Every call to db)set appends to the emd of the file, so if you update a key serveral times, the old versions of the value are not overwritten - you need to look at the last occurence of a key in a file to find the latest value (hence the tail -n 1)  

On the other hand, our db_get function has terrible performance if you have a large number of records in your database. Everytime you want to look up a key, db_get has to scan the entire database file from begining to end, looking for occurrences of the key. In algoirthmic terms, the cost of a lookup is O(n): if you double the number of records n in your database, a lookup takes twitce as long. That is not good.  

In order to efficiently find the value for a particular key in a database, we need a different data structure: an index. In this chapter we will look at a range of indexing structures and see how they compare; the general idea behind them is to keep some additional metadata on the side, which acts as a signpost and helps you locate the data oyu want. If you want to search the data in several different ways, you may need serveral different indexes on different parts of the data.  

An index is an additional structure that is derived from the primary data. Many databases allow you to add and remove indexes, and this does not affect the contents of the databasel it only affects the performance of queries. Maintaining additional structures incurs overhead, especially on writes. For writes, it is hard to beat the performance of simply appending to a file, because that is the simplest possible write operation. Any kind of index usually slows down writes, because the index also needs to be updated every time data is written.  

This is an important trade0off in storage sysyems: well-chosen indexes speed up read queries, but every index slows down writes. For this reason, databases do not usually index everything by default, but requires- the application developer or database administrator to choose indexes manually, using your knowledge of applications typical query patterns. You can them choose the indexes that give your application the greatest benefit, without introducing more overhead than necessary. 

### Hash indexes

Let's start with indexes for key-value data. This is not the only kind of data you can index, but it is very common, and it is a useful building block for more complex indexes.  

Key-value stores are quite similar to the dictionary type that you can find in most programming languages, and which is usually implemented as a hash map. Hash maps are described in many algorithms textbooks, so we won;t go into detail of how they work here. Since we already have hash maps for your inmemory data structures, why not use them to index you data on disk?  

Let's say our data storage consists only of appends to a file, as in the preceding example. Then the simplest possible indexing strategy is this: keep an in-memory jash map where every key is mapped to a byte offset in the data file - the location which the value can be found. Whenever you append a new key-value pair to the file, you also update the hhash map to reflect the offset of the data you just wrote (this works both for inserting new keys and for updating existing keys). When you want to look up a value, use the hash map to find the offset in the data file, seek to that ocation and read the value.  

This may sound simplistic, but it is a viable approach. In fact, this is essentially what Bitcask does. Bitcask offers high=performance reads and writes, subject to the requirement that all the keys fit in the available RAM, since the hash map is kept completely in memory. The values can use more spcae than there is available memory, since they can be loaded from disk with just one disk seek. If part of the data file is already in the filesysten cachem a read does not requre any disk I/O at all.  

A storage engine like Bitcask is well suited to situations where the value for each key is updated frequently. For example, the key might be the URL of a cat video, and the value might be the number of times it has been plyed (incremented every time someone hits the play button). In this kind of workload, there are a lot of writes, but there are not too many distinct keys - you have a large number of writes per key, but there are not too many distinc keys - you have a large number of writes per key, but it is feasible to keep all keys in memory.  

As described so far, we only ever append to a file - so how do we avoid eventually running out of disk space? A good solution is to break the log into segments of a certain size by closing a segment file when it reaches a certain size, and making subsequent writes to a new segment file. We can then perform compaction on these segments. Compaction means throwing away duplicate keys in the log, and keeping only the most recent update for each key.  

example of compaction:  
A data file segment with repeated names and updated values. Only taking the last occurence of the name with its updated value.  

Moreover, since compaction often makes segments much smaller (assuming that a key is overwritten serveral times on average within on segment), we can also merge serveral segments together at the same time as performing compaction. Segments are never modified after they have been written, so the merged segment is written to a new file. The merging and compaction of frozen segments can be done in a background thread, and while it is going on, we can still continue to serve read and write requests as normal, using the old segment files. After the merging process is complete, we switch read requests to using the new merged segment instead of the old segments - and the old segment files can simply be deleted.  

Each segment now has its own in-memory hash table, mapping keys to file offsets. In order to find the value for a key, we first check the most recent segment's hash map; if the key is not present we check the second-most recent segment, and so on. The merging process keps the number of segments small, so lookups do not need to check many hash maps.  

Lots of details goes into making this simple idea work in practice. Brieflym some of the issues are that are important in real implementation are:  

File format:  
    - CSV is not the best format for a log. It is faster and simpler to use a binary format that encodes the length of a string in bytes, followed by the raw string (without need for escaping).  
    
Deleting records:  
    - If you want to delete a key and its associated valuem you ahve to append a special deletion record to the data file (sometimes called a tombstone). When a log segments are merged, the tombstone tells the merging process to discard any previous values for the deleted key.  
    
Crash recovery:  
    - If the database is restarted, the in-memory hash maps are lost. In prinicple, you can restore each segment's hash map by reading the entire segment file from beginning to end and noting the offset of the most recent value for every key you go along. However, this migh take a long time if the segment files are large, which would make server restarts painful. Bitcask speeds up recovery by storing a snapshot of each segment's hash map on disk, which can be loaded into memory more quickly.  
    
Partially written records:  
    - The database may crash at any time, including halfway through appending a record to the log. Bitcask files include checksums, allowing such corrupted parts of the log to be detected and ignored.  
    
Concurrency control:  
    - As writes are appended to the log in a strictly sequential order, a common implementation choice is to have only one writer thread. Data file segments are append-only and otherwise immutable, so they can be read concurrently by multiple threads.  
    
An append-only log seems wasteful at first glance: why don't you update the file in place, overwriting the old value with the new value? But an append-only design turns out to be good for serveral reasons:  

    - Appending and segment merging are sequential write operations, which are generally much faster than random writes, especially on magnetic spinning-disk hard drives To some extent sequential writes are also preferable on flash-based solid state drives (SSDs). We will discuss this issue in later in 'comparing B-Trees and LSM-Trees.'  
    
    - Concurrency and crash recovery are much simpler if segment files are append only or immutable. For example, you don not have to worry about the case where a crash happened while a value was being overwritten, leaving you with a file containing part of the old and part of the new value spliced together.  
    
    - merging old segments avoids the problem of data files getting framgented over time.  
    
However, the hash table index also has limitations:  

    - The hash table must fit in memory, so if you have a very large number of keys, you are out of luck. In principle, you could maintain a hash map on disk, but unfortunately it is difficult to make an on-disk hash map perform well. It requires a lot of random access I/O, it is expensive to grow when it becomes full, and hash collisions requre fiddly logic.  
    
    - Range queries are not efficient. For example, you cannot easily scan over all keys between kitty00000 and kitty99999 - you would have to look up each key individually in the hash maps.  
    
In the next section we will look at an indexing structure that does not have those limitations.  

### SSTables and LSM-Trees


In Figure 3-3, each log-structured storage segment is a sequence of key-value pairs. These pairs appear in the order that they were written, and value later in the log take precedence over values for the same key earlier in the log. Apart from that, the order of key-value pairs inthe file does not matter.  

Now we can make a simple change to the format of your segment files: we require that the sequence of key-value pairs is sorted by key. At first glance, that requirement seems to break our ability to use sequential writes, but we will get to that in a moment.  

We call this format Sorted String Table, or SSTable for short. We also require that each key only appears once within each merged segment file (the compaction process already ensures that). SSTables ahave several big advantages over log segments with hash indexes:  

1. Merging segments is simple and efficient, even if the files are bigger than the available memory. The approach is like the one used in the mergesort algorithm and is illustrated in Figure 3-4: you start reading the input files side by side, look at the first key in each file, copy the lowest key (according to the sort order) to the output file, and repeat. This produces a new merged segment file, also sorted by key.  

What if the same key appears in several input segments? Remember that each segment contains all the values written to the database during some period of time. This means that all the values in one input segment must be more recent than all the values in the other segment (assuming gthat we always merge adjacent segments). When mutliple segments contain the same key, we can keep the value from the most recent segment and discard the values in older segments.  

2. In order to find a particular key in the file, you no longer need to keep an index of all the keys in memory. See Figure 3-5 for an example: say you are looking for the key handiwork, but you do not know the exact offset of that key in the segment file. However, yuo do know the offset of the keys handbag and handsome, and because of the sorting you kinow that the key handiwork must eppear between those two. This means you can jump to the offset for handbag and scan from there unti you find handiwork (or not, if the key is not present in the file).  

You still need an in memory index to tell you the offset for some of the keys, but it can be sparse: one key for every few kilobytes of segment file is sufficient, because a few kilobytes can be scanned very quickly.  

3. Since read requests need to scan over several key-value pairs in the requested range anyway, it is possible to group those records into a block and compress it before writing it to disk (indicated by the shaded area in Figure 3-5). Each entry of the sparse in-memory index then points at the start of a compression block. Besides saving disk space, compression also reduces the I/O badnwdith use.  

### Constructing and maintaining SSTables  

Fine so far - but how do you get your data to be sorted by key in the first place? Our incoming writes can occurn in any order.  

Maintaining a sorted structure on disk is possible (with B-trees), but maintaining it in memory is much easier. There are plenty of well-known tree data structures that you can use, such as red-black trees or AVL trees. With these data structures, you can insert keys in any order and read them back in sorted order.  

We can now make our storage engine work as follows:  

- When a write comes in, add it to an in-memory balanced tree data structure (for example, a red-black tree). This in-memory tree is sometimes called a memtable.  
- When the memtable gets bigger than some threshold - typically a few megabytes - write out to a disk as an SSTabe file. This can be done efficiently because the tree already maintains the key-value pairs sorted by key. The new SSTable file becomes the most recent segment of the database. While the SSTable is being written out to disk, writes can continue to a new memtable instance.  
- In order to serve a read request, first try to find the key in the memtable, then in the msot recent on-disk segment, then in the next-older segment, etc.  
- From time to time, run a merging and compaction process in the background to combine segment files and to discard overwritten or deleted values.  

This scheme works very well. It only suffers from one problem: if the database crashes, the most recent writes (which are in the memtable but not yet written out to disk) are lost. In order to avoid that problem, we can keep a separate log on disk to which every write is immediately appended, just like in the previous section. That log is not in sorted order, but that does not matter, because it its only purpose is to restore the memtable after a crash. Every time the memtable is written out to an SSTable, the corresponding log can be discarded.  

### Making an LSM-tree out of SSTables  

The algorithm described here is essentially what is used in LevelDB and ROcksDB, key-value engine libraries that are designed to be embedded into other applications. AMong other things, LEvelDB can be used in Riak as an alternative to Bitcask. Similar storage engines are used in Cassandra and HBase, both of which were inspired by Google's Bigtable papyer (which introduced the terms SSTable and memtable).  

Originally this indexing structure was described by Patrick O'Neil et al. under the name Log-Structured Merge-Tree (or LSM-Tree), building on earlier work on log-structued fiel systems. Storage engines that are based on this principle of merging and compacting sorted files are often called LSM storage engines.  

Lucene, an indexing engine for full-text search used by Elasticsearch and Solr, uses a similar method for storing ints term dictionary. A full=text indexing engine for full-text search used by Elasticsearch and Solr, uses a similar method for storing its term dictionary. A full-text index is much more complex than a key-value index but is based on a similar idea: given a word in a search query, find all the documents (web pages, product descriptions, etc.) that mention the word. This is implemented with a key-value structure where the key is a word (a term) and the value is the list of IDs of all the documents that contain the word ( the posting list). In Lucene, this mapping from term to posting list is kept in SSTable-like stored files, which are merged in the background as needed.  

### Performance optimizations  

As always, a lot of detail goes into making a storage engine perform well in practive, For example, the LSM-tree algorithm can be slow when looking up keys that do not exist in the database: you ahve to check the memtable, then the segments all the way back to the oldest (possibly having to read from disk for each one) before you can be sure that the key does not exist. In order to optimize this kind of access, storage engines often use additional Bloom filters. (A Bloom filter is a memory-efficient data structure for approximating the contents of a set. It can tell you if a key does not appear in the database, and thus saves many unnecessary disk reads for nonexistent keys.)  

There are also different strategies to determine the order and timing of how SSTables are compacted and merged. The most common options are size-tiered and leveled compaction. LevelDB and RocksDB use leveled compaction (hence the name of LevelDB), HBase uses size-tiered, and Cassandra supports both. In size-tiered compaction, newer and smaller SSTables are successively merged into older and larger SSTables. In leveled compaction, the key range is split up into smaller SSTables and older data is moved into separate 'levels', which allows the compaction to proceed more incrementally and use less disk space.  

Even though there are many subtleties, the basic idea of LSM-trees is keeping a cascade of SSTables that are merged in the background -- is simple and effetive. Even when the dataset is much bigger than the available memory it continues to work well. Since data is stored in sorted order, you can efficiently perform range queries (scanning all keys above some minimum and up to some maximum), and because the disk writes are sequential the LSM-tree can support remarkably high write throughput.  

### B-Trees

The log-structured indexes we have discussed so far are gaining acceptance, they they are not the most common type of index. The most widely used indexing sturcture is quite different: the B-tree.  

Introduced in 1970 and called "ubiquitous" less than 10 years later, B-trees have stood the test of time very well. They remain the standard index implementation in almost all relational databases, and many nonrelational databases use them too.  

Like SSTables, B-trees keep key-value pairs sorted by key, which allows efficient key-value lookups and range queries. But that's where the similarities endL B-trees have a very different design philosophy.  

The log-structued indexes we saw earlier break the database down into variable-size segments, typically several megabytes or more in size, and always write a segment sequentially. By contrast, B-trees break the database down into fixed-size blocks or pages, traditionally 4 KB in size (sometimes bigger), and read or write on page at a time. This design corresponds more closely to underlying hardware, as disks are also arranged in fixed-size blocks.  

Each page can be identified using an address or location, which allows one page to refer to another -- similar to a pointer, but on disk instead of in memory. We can use those page references to construct a tree of pages, as illustrade in Figure 3-6.  

One page is designed as the root of the B-tree; whenever you want to look up a key in the index, you start here. The page contains several keys and references to child pages. Each child is responsible for a continouous range of keys, and the keys between the references indicate where the boundaries between those ranges lie.  

In the example in Figure 3-6, we are looking for the key 251, so we know that we need to follow the page references between the boundaries 200 and 300. That takes us to a similar-looking page that further breaks down the 200-300 range into subranges.  

Eventually we get down to a page containing individual keys (a leaf page), which either contains the value for each key inline or contains a reference to the page where the values can be found.  

The number of references to  child pages in one page of the B-tree is called the branching factor. For example, in Figure 3-6 the branching factor is six. In practice, the branching factor depends on the amount of space required to store the page references and the range boundaries, but typically it is several hundred.  

If you want to update the value for an existing key in a B-tree, you search for the leaf page containing tht key, change the value in that page, and write the page back to disk (any references to that page remain valid). If you want to add a new key, you need to find the page whose range encompasses the new key and add it to that page. If there is not enough free space in the page to accommodate the new key, it is split into two half-full pages, and the parent page is updated to account for the new subdivision of key ranges -- see Figure 3-7.  

This algorithm ensures that the tree remains balanced: a B-tree with n keys always has a depth of O(log n). Most databases can fit into a B-tree that is three or four levels deep, so you do not need to follow many page references to find the page you are looking for. (A four level tree of 4KB page with a branching factor of 500 can store up to 256 TB).  

### Making B-trees reliable  

The basic underlying write operation of a B-tree is to overwrite a page on disk with new data. It is assumed that the overwrite does not change the location of the page; i.e., all references to that page remain intact when the page is overwritten. This is in stark constrast to log-structured indexes such as LSM-trees, which only append to files (and eventually delete obsolete files) but never modify files in place.  

You can think of overwriting a page on disk as an actual hardware operation. ON a magnetic hard drive, this means moving the disk head to the right place, waiting for the right position on the spinning plater to come around, and the overwritting the appropriate sector with new data. On SSDs, what happens is somewhat more complicated, due to the fact that an SSD must erase and rewrite fairly large blocks of a storage chip at a time.  

Moreover, some operations requre several different pages to be overwritten. For example, if you split a page because an insertion caused it to be overfull, you need to write the two pages that were splitm and also overwrite their parent page to update the references to the two child pages. This is a dangerous operation, because if the database crashes after only ome of the pages have been written, you end up with a corupted index (e.g., there may be an rphan page that is not a child of any parent).  

In order to make the database resilient to crashes, it is common for B-tree implementations to include an additional data structure on disk: a write-ahead log (WAL, also known as a redo log). This is an append-only file to which every B-tree modification must be written before it can be applied to the page of the tree itself. WHen the database comes back up after a crash, this log is used to restore the B-tree back to a consistent state.  

An additional complication of updating pages in place is that careful concurrency control is required if multiple threads are going to access the B-tree at the same time -- otherwise a thread may see the tree in an inconsistent state. This is typically done by protecting the tree's data structure with latches (lightweight locks). Logstructured approaches are simplier in this regard, because they do all the merging in the background without interfering with incoming queries and atomically swap old segments for new segments from time to time.  

### B-tree optimizations  

As B-trees have been around for so long, it is not surprising that many optimizations have been developed over the years. To mention a few:  

- instead of overwriting pages and maintaining a WAL for crash recovery, some databases (like LMDB) use a copy-on-write scheme. A modified page is written to a different location, and a new version of the parent page in the tree is created, pointing at the new location. This approach is also useful for concurrency control, as we shall see in 'Snapshot Isolation and Repeatable Read' on page 237.  
- we can save space in pages by not storing the entire key, but abbreviating it. Especially in pages on the interior of the tree, keys only need to provide enough information to act as boundaries between key ranges. Packing more keys into a pageallows the tree to have a higher branching factor, thus fewer levels.  
- In general, pages can be posiioned anywhere on the disk; there is nothing requiring pages with nearby key ranges to be nearby on disk. If a query needs to scan over a large part of the key range in sorted order, that page-by-page layout can be inefficient, because a disk seek may be required for every page that is read. Many B-tree implementations therefore try to lay out the tree so that leaf pages appear in sequential order on disk. However, it is difficult to maintain that order as the tree grows. By contrast, since LSM-trees rewrite large segments of the storage in on go during merging, it is easier for them to keep sequential keys close to each other on disk.  
- Additional pointers have been added to the tree. For example, each leaf page may reference to its sibling pages to the left and right, which allows scanning keys in order without jumping back to parent pages.  
- B-tree variants such as fractal trees borrow some log-structured ideas to reduce disk seeks (and they have nothing to do with fractals).  

### Comparing B-Trees and LSM-Trees

Even though B-tree implementations are generally more mature than LSM-tree implementations are generally more mature than LSM-tree implementations, LSM-trees are also interesting due to their performance characteristics. As a rule of thumb, LSM-trees are typically faster for writes, whereas B-trees are thought to be faster for reads. Read are typically slower on LSM-trees because they have to check several different data structures and SSTables at different stages of compaction.  

However, benchmarks are often inconclusive and sensitive to details of the workload. You need to test systems with your particular workload in order to make a valid comparison. In this section we will briefly discuss a few things that are worth considering when measuring the performance of a storage engine.  

### Advantages of LSM-trees 

A B-tree index must write every piece of data at least twice: once to the write-ahead log, and once to the tree page itself (and perhaps again as pages are split). There is also overhead from having to write an entire page at a time, even if only a few bytes in that page changed. Some storage engines even overwrite the same page twitce in order to avoid ending up with a partially updated page in the even of a power failure.  

Log-structued indexes also rewrite data multiple times due to repeated compaction and merging of SSTables. This effect -- one write to the database resulting in multiple writes to the disk over the course of the database's lifetime -- is known as write amplification. It is of particular concern on SSDs, which can only overwrite blocks a limited number of times before wearing out.  

In write-heavy applications, the performance bottleneck might be the rate at which the database can write to disk. In this case, write amplification has a direct performance cost: the more that a storage engine writes to disk, the fewer writes per second it can handle within the available disk bandwidth.  

Moreover, LSM-trees are typically able to sustain higher write throughput thab B-trees, partly because they sometimes have lower write amplification (although this depends on the storage engine configuration and workload), abd partly because they sequentially write compact SSTable files rather than having to overwrite several pages in the tree. This difference is particularly important on magnetic hard drives, where sequential writes are much faster than random writes.  

LSM-trees can be compreesed better, thus often produce smaller files on disk than B-trees. B-tree storage engines leave some disk space unused due to fragmentationL when a page is split or when a row cannot fit into an existing page, some space in a page remains unused. Since LSM-treesa re not page-priented and periodically reqrite SSTables to remove fragmentation, they have lower storage overheads, especially when useing leveled compaction.  

On many SSDs, the firmware internally uses a log-structured algoirthm to turn random writes into sequential writes on the underlying storage chips, so the impact of the storage engine's write pattern is less pronounced. However, lower write amplification and reduced fragmentation are still advantageous on SSDs: representing data more compactly allows more read and write requests within the available I/O bandwidth.  

### Downside of LSM-trees 

A downside of log-structured storage is that the compaction process can sometimes interfere with the performance of ongoing reads and writes. Even though storage engines try to perform compaction incrementally and without affecting concurrent access, disk have limited resources, so it can easily happen that a request needs to wait while the disk finishes an expensive compaction operation. The impact on throughput and average response time is usually small, but at higher percentiles the response time of queries to log-structured storages engines can seomtimes be quite high, and B-trees can be more predictable.  

Another issue with compaction arises at high write throughput: the disk's finite write bandwidth needs to be shared between the initial write (logging and flushing a memtable to disk) and the comapction threads running in the background. When writing to an empty database, the full disk bandwidth can be used for the initial write, but the bigger the database gets, the more disk bandwidth is required for compaction.  

If write throughput is high and compaction is not configured carefully, it can happen that compaction cannot keepe up with the rate of incoming writes. In this case, the number of unmerged segments on disk keeps growing until you run out of disk space, and reads also slow down because they need to check more segment files. Typically, SSTable-based storage engines do not throttle the rate of incoming writes, even if compaction cannot keep up, so you need explicit monitoring to detect this situation.  

An advanage of B-trees is that each key exists in exactly one place in the index, whereas log-structured storage engine may have multiple copies of the same key in different segments. This aspect makes B-trees attractive in databases that want to offer strong transactional semantics: in many relational databases, transactional isolation is implemented using locks on ranges of keys, and in a B-tree index, those locks can be directly attached to the tree. In chapter 7 we will discuss this point in more detail.  

B-trees are very ingrained in the architecture of databases and provde consistently good performance for many workloads, so it is unlikely that they will go away anytime soon. In new datastores, log-structured indexes are becoming increasingly popular. There is no quick and easy rule for determining which type of storage engine is better for your use case, so it is worth testing empirically.  

### Other indexing structues  

So far we have only discussed key-value indexesm which are like a primary key index in the relational model. A primary key uniquely identifies one row in a relational table, or one document in  document database, or one vertex in a graph database. Other records in the database can refer to the row/document/vertex by its primary key (or ID), and the index is used to resolve such references.  

It is also very common to have secondary indexes. In relational databases, you can create several secondary indexes on the same table using the CREATE INDEX command, and they are often crucial for performing joins efficiently. For example, in Figure 2-1 in chapter 2 you would most likely have a secondary index on the user_id columns so that you can find all the rows belonging to the same user in each of the tables.  

A secondary index can easily be constructued from a key-value index. The main difference is that keys are not unique; i.e., there might be many rows (documents, verticies) with the same key. This can be solved in two ways: either by making each value in the index a list of matching row identifiers (like a posting list in a full-text index) or by making each key unique by appending a row identifier to it. Either way, both B-trees and log-structured indexes can be used as secondary indexes.  

### Storing values within the index  

The key in an index is the thing that queries search for, but the values can be one of two things: it can be the actual row(document, vertex) in question, or it could be a reference to the row stored elsewhere. In the latter case, the place where rows are stored is known as a heap file, and it stores data in no particular order (it may be append-only or it may keep track of deleted rows in order to overwrite them with new data later). The heap file approach is common because it avoids duplicating data when multiple secondary indexes are present: each index just references a location in the heap file, and the actual data is kept in one palce.  

When updating a value without changing the key, the heap file approach can be quite efficient: the record can be overwritten in place, provided that the new value is not larger than the old value. The situation is more complicated if the new value is larger, as it probably needs to be moved to a new location in the heap where there is enough space. In that case, either all indexes need to be updated to point at the new heap loation of the record, or a forwarding pointer is left behind in the old heap location.  

In some situations, the extra hop from the index to the heap file is too much of a performance penalty for reads, so it can be desirable to store the indexed row directily within an index. This is known as clustered index. For example, in MySQL's InnoDB storage engine, the primary key of a table is always a clustered index, and secondary indexes refer to the primary key (rather than a heap file location). In SQL Server, you can specify one clustetered index per table.  

A compromise between a clustered index (storing all row data within the index ) and a nonclustered index (storing only references to the data within the index) is known as a covering index or index with included columns, which stores some of a table's columns within the index. This allows some queries to be answered by using the index alone (in which case, the index is said to be covered the query).  

As with any kind of duplication of data, clustered and covering indexes can speed up reads, but they requre additional storage and can add overhead on writes. Databases also need to go to additional effort to enforce transactional guarantees, because applications should not see inconsistencies due to the duplication.  

### Multi-column indexes  

The indexes discussed so far only map a single key to a value.. That is not sufficient if we need to query mutiple columns of a table (or multiple fields in a document) simultanuously.  

The most common type of multi-column index is called a concatenated index, which simply combines several fields into one key by appending one column to another (the index definition specifies in which order the fields are concatentated). This is like an old-fashioned paper phone book, which provides an index from (lastname, firstname) to phone number. Due to the sort order, hte index can be used to find all the people with a particular last name, or all the people with a particular lastname-firstname combination. However, the index is useless if you want to find all the people with a particular fire name.  

Multi-dimensional indexes are a more general way of querying several columns at once, which is particulary important for geospatial data. For example, a restaurant search website may have a database containing the latitude and longitude of each restaurant. When a user is looking at the restaurants on a map, the website needs to search for all the restaurants within the rectangular map area that the user is currently viewing. This requires a two-dimensional range query like this following:  

SELECT * FROM RESTAURANTS  
WHERE latitude > 51.444 AND latitude < 51.5555  
AND longitude > -0.1111 AND latitude < -.01004;  

A standard B-tree or LSM-tree index is not able to answer that kind of query efficiently: it can give you either all the restaurants in range of latitudes (but any longitude), or all the the restaurants in range of longitude (but anywhere between the North and South poles), but not both simultaneously.  

One option is to translate a two-dimensional location into a single number using a space-filing curve, and then to use a regular B-tree index. More commonly, specialized spatial indexes such as R-trees are used. For example, PostGIT implements geospatial indexes as R-trees using PostgreSQL's Generalized Search Tree indexing facility. We don't have space to describe R-trees in detail here, but there is plenty of literature on them.  

An interesting idea is that multi-dimensional indexes are not jsut for geographic locations. For example, on an ecommerce website you could use a three-dimensional index on dimensions (red, green, blue) to search for products in a certain range of colors, or in a database of weather observations you could have a two-dimensional index on (date, temperature) in order to efficiently search for all the obsevations during the year 2013 where the temperature was between 25 and 30 Celcius Degrees. With a one-dimensional index, you would have to either scan over all the records from 2013 (regardless of temperature) and then filter them by temperature, or vice versa. A 2D index could narrow down by timestamp and temperature simulatneously. This teachnique is used by HyperDex.  

### Full-text search and fuzzy indexes  

All the indexes discussed so far assume that you have exact dat and allow you to query for exact values of a key, or range of values of key with a sort order. What they do not allow you to do is search for similar keys, such as misspelled words. Such fizzy querying requires different techniques.  

For example, full-text search engines commonly allow a search for one word to be expanded to include synonyms of the word, to ignore grammatical variations of words, and to search for occurerences of words near each other in the same document, and support various other features that depend on linguistic analysis of the text. To cope with typos in documents or queries, Lucene is able to search text for words within a certain edit distance (an edit distance of 1 means that one letter has been added, removed, or replaced).  

As mentioned in 'Making an LSM-tree out of SSTables', Lucene uses a SSTable-like structure for its term dictionary. This structure requires a small inmemory index that tells wueries at which offset in the sorted file they need to look for a key. In LevelDB, this in-memory index is a sparse collection fo some of the keys, but in Lucene, the in-memory index is a finite state automaton over the characters in the keys, similar to 'rie'. This automaton can be transformed into a 'levenshtein automaton', which supports efficient search for words within a given edit distance.  

Other fuzzy search techniques go inthe direction of document classification and machine learning. See an information retrieval textbook for more detal.  

### Keeping everything in memory  

The data structures discuessed so far in this chapter have all been answered to the limitations of disks. Compared to main memory, disks are awkward to deal with. With both magnetic disks and SSDs, data on disk needs to be laid out carefully if you want good performance on reads and writes. However, we tolerate this awkwardness because disks have two significant advantagesL they are durable *their contents are not lost if the power is turned off), and they have a lower cost per gigabyte than RAM.  

As RAM becomes cheaper, the cost-per-gigabyte argument is eroded. Many datasets are simply not that big, so it is quite feasible to keep them entirely in memory, potentially distributed across several machines. This has led to the development of inmemory databases.  

Some in-memory key-value stores, such as Memcached, are inteded for caching use only, where it is acceptable for data to be lost if a machine is restarted. But other inmemory databases aim for durability, which can be achieved with special hardware (such as battery-powered RAM), by writing a log of changes to disk, by writing periodic snapshots to disk, or by replicating the in-memory state to other machines.  

When an in-memory database is restarted, it needws to reload its state, either from disk or over the network from replic (unless special hardware is used). Despite writing to disk, it is still an in-memory database, because the disk is merely used as an append-only log for durability, and reads are served entirely from memory. Writing to disk also has operational advantages: files on disk can easily be backed up, inspected, and analyzed by external utilities.  

Products such as VoltDB, MemSQL, and Oracle TimesTen are in-memory database with a relational model, and the vendors claim that they can offere big performance improvements by removing all the overheads associated with managing on-disk data structures. RAMCloud is an open source, in-memory key-value store with durability (using a log-structyred approach for the data in memory as well as the data on disk). Redis and Couchbase provide weak durability by wriiting to disk asynchronously.  

Counterintuitively, the performance advantage of in-memory databases in not due to the fact that they don't need to read from disk. Even a disk-based storage engine may never need to read from disk if you have enough memory, because the operating system caches recently used disk blocks in memory anyway. Rather, they can be faster because they can avoid the overheads of encoding in-memory data structure in form tht can be written to disk.  

Besides performance, another interesting area for in-memory databases is pr oviding data models that are difficult to implement with disk-based indexes. For example, Redis offeres a database-like interface to various data structures such as priority queues and sets. Because it keeps all data in memory, its implementation is comparatively simple.  

Recent research indicates that an in-memory architecture could be extended to support datasets larger than the available memory, without bringing back the over heads of a disk-centric architectures. This so-called anti-caching approach works by evicting the least recently used data from meory to disk when there is not enough memory, and loading it back into memory when it is accessed again in the future. This is similar to what operating systems do with citrual memory and swap files, but the database can manage memory more fficiently than the OS, as it can work at the granularity of individual records rather than entire memory pages. THis approach still requires indexes to fit entirely in memory, though (like the Bitcask example at the beginning of the chapter).  

Further changes to storage engine design will probably be needed if non-volatile memory (NVM) technologies become more widely adopted. At present, this is a new area of research, but it is worth keeping an eye on in the future.  

### Transaction Processing or Analytics?  



# Notes

I learned that though a DE will not be implementing a custom database storage and retrieval internally, you need to have some understanding to select an appropriate storage engine for your application. This will also allow you to tune the storage engine to perform better given the type of expected workload.

The author created a simple key value database that appends. He showed that for lookup(read) it is a O(n) since when you look up a key you have to scan the entire database file from begining to end. 

To efficiently find a value for a key in a database, a different data structure containing an index is used. An index keeps metadata on the side, and helps you locate the data you want. Indexes can be added which affects the performance of the queries with the trade-off of a slower write but a faster read. Databases don't usually index everything by default, but require developers or data admins to choose indexes to give the greatest performance benifit without introducing more overhead than necessary.  
