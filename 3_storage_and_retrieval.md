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



# Notes

I learned that though a DE will not be implementing a custom database storage and retrieval internally, you need to have some understanding to select an appropriate storage engine for your application. This will also allow you to tune the storage engine to perform better given the type of expected workload.

The author created a simple key value database that appends. He showed that for lookup(read) it is a O(n) since when you look up a key you have to scan the entire database file from begining to end. 

To efficiently find a value for a key in a database, a different data structure containing an index is used. An index keeps metadata on the side, and helps you locate the data you want. Indexes can be added which affects the performance of the queries with the trade-off of a slower write but a faster read. Databases don't usually index everything by default, but require developers or data admins to choose indexes to give the greatest performance benifit without introducing more overhead than necessary.  

