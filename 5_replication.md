# Replication

Replication means keeping a copy of the same data on multiple machines that are connected via a network. As discussed in the introduction in Part 2, there are several reasons why you might want to replicate data:  

- To keep data geographically close to your users (and thus reduce latency)
- To allow a system to continue working even if some of its parts have failed (and thus increase availability)
- To scale out the number of machines that can server read queries (and thus increase read throughput)

In this chapter we will assume that your dataset is so small that each machine can hold a copy of the entire dataset. In Chapter 6 we will relax that assumption and discuss partitioning (sharing) of datasets that are too big for a single machine. In later chapters we will discuss various kinds of faults that can occur in a replicated data system, and how to deal with them. 

If the data you're replicaing does not change over time, then replicating is easy: you just need to copy the data to every node once, and you're done. All of the difficulty in replication lies in handling changes to replicated data, and that's what this chapter is about. We will discuss three popular algorithms for replicating changes between nodes: single-leader, multi-leader, and leaderless replication. Almost all distributed databases use one of these three approaches. They all have various pros and cons, which we will examine in detail.  

There are many trade-offs to consider with replication: for example, wheter to use synchronous or asynchronous replication, and how to handle failed replicas. Those are often configuration options in databases, and although the details vary by database, the general principles are similar across many different implementations. We will discuss the consequences of such choices in the chapter.  

Replication of databases is an old topic -- the principles haven't changed much since they were studied in the 1970s, because the fundamental constants of networks have remained the same. Howeverk outside of research, many developers continued to assume for a long time that database consisted of just one node. Mainstream use of distributed databases is more recent. Since many application developers are new to the area, there has been a lot of misunderstanding around issues such as eventual consistency. In "Problems with Replication Lag" on page 161 we will get more precise about event consistency and discuss things like the read-your-writes and monotonic reads guarantees.  

# Leaders and Followers  

Each node that stories a copy of the database is called a replica. With mulitple replicas, a question inevitably arises: how do we ensure that all the data ends up on all the replicas?  

Every write to the database needs to be process by every replica; otherwise, the replicas would no longer contain the same data. The most common solution for this is called leader-based replication (also known as active/passive of master-slave replication) and is illustrated in Figure 5-1. It works as follows:  

1. One of the replicas is designated the leader (also known as master or primary). When clients want to write to the database, they must send their requests to the leader, which first writes the new data to its local storage.  

2. The other replicas are known as followers (read replicas, slaves, secondaries, or hot standbys). Whenever the leader writes new data to its local storage, it also sends the data change to all of its followers as part of replication log or change stream. Each follower takes the log from the leader and updates its local copty of the database accordingly, by applying all the writes in the same order as they were processed on the leader.  

3. When a client wants to read from the database, it can query either the leader or any of the followers. However, writes are only accepted on the leader (the followers are read-only from the client's point of view).  

This mode of replication is built-in feature of many relational databases, such as PostgreSQL, MySQL, Oracle Data Gyard, and SQL Server's AlwaysOn Availability Groups. It is also used in some nonrelational databases, including MongoDB, RethinkDB, and Espresso. Finally, leader-based replication is not restricted to only databases: distributed message brokers such as Kafka and RabbitMQ highly available queues also use it. Some network filesystems and replicated block devices such as DRBD are similar.  

# Synchronous Versus Asynchronous Replication

An important detail of replicated systems is whether the replication happens synchronously or asynchronously. (in relational databases, this is often a configurable option; other systems are often hardcoded to be either one or the other.)  

Think about what hapens in Figure 5-1, where the user of a website updates their profile image. At some point in time, the client sends the update request to the leader; shortly afterward, it is received by the leader. At some point, the leader forwards the data change to the followers. Eventually, the leader notifies the client that the update was succesful.  

Figure 5-2 shows the communication between various components of the system: the user's client, the leader, and two followers. Time flows from left to right. A request or response message is shown as a thick arrow.  

In the example of Figure 5-2, the replication to follower 1 is synchronous: the leader waits until follower 1 has confirmed that it received the write before reporting success to the user, and before making the write visible to other client. The replication to follower 2 is asychronous: the leader sends the message, but doesn't wait for a response from the follower.  

The diagram shows that there is substantial delay before followers 2 processes the message. Normally, replication is quite fast: most database system apply changes to followers in less than a second. However, there is no guarantee of how long it might take. There are circumstances when followers might fall behind the leader by serveral minutes or more; for example, if a follower is recovering from a failure, if the system is operating near maximum capacity, or if there are network problems between the nodes.  

The advantage of synchronous replication is that the follower is guaranteed to have an up-to-date copy of the data that is consisten with the leader. If the leader suddenly fails, we can be sure that the data is still available on the follower. The disavantage is that if the synchronous follower doesn't respond (because it has crashed, or there is a network fault, or for any other reason), the write cannot be processed. The leader must block all writes and wait until the synchronous replica is available again.  

For that reason, it is impractical for all followers to be synchronous: any one node outage would cause the whole system to grind to a halt. In practice, if you enable synchronous replication on a database, it usually means that one of the followers is synchronous, and the others are asynchronous. If the synchronous follower becomes unavailable or slow, one of the asynchronous followers is made synchronous. This guarantees that you have an up-to-date copy of the data on at least two nodes: the leader and one synchronous follower. The configuration is sometimes called semi-synchronous.  

Often, leader-based replication is configured to be completely asynchronous. In this case, if the leader fails and is not recovereable, any writes that have not yet been replicated to followers are lost. This means that a write is not guaranteed to be durable, even if it has been confirmed to the client. However, a fully asynchronous configuration has the advantage that the leader can continue processing writes, even if all of its followers have fallen behind.  

Weakening durability may sound like a bad trade-off, but asynchronous replication is neverthelss widely used, especially if there are many followers or if they are geographically distributed. We will return to this issue in "Problems with Replication Lag" on page 161.  

# Research on Replication

It can be a serious problem for asynchronous replicated systems to lose data if the leader fails, so researchers have continued investigating replication methods that do not lose data but still provide good performane and availability. For example, chain replication is a variant of synchronous replication that has been successfully implemented in a few systems such as Microsoft Azure Storage.  

There is strong connection between consistency of replication and consensus (getting several nodes to agree on a value), and we will explore this area of theory in more detil in Chapter 9. In this chapter we will concentrate on the simpler forms of replication that are most commonly used in databases in practice.  

# Setting Up New Followers 

From time to time, you need to set up new followers -- perhaps to increase the number of replicas, or to replace failed nodes. How do you ensure that the new followers has an accurate copy of the leader's data?  

Simply copying data files from one nodes to another is typically not sufficient: client are constantly writting to the database, and the data is always in flux, so a standard file copy would see different parts of the database at different points in time. The result might not make any sense.  

You could make the file on disk consisten by locking the database (making it unavailable for writes), but that would go against our goal of high availability. Fortunately, setting up a follower can usually be done without downtime. Conceptually, the process looks like this:  

1. Take a consistent snapshot of the leader's database at some point in time - if possible, without taking a lock on the entire database. Most database have this feature, as it is also required for backups. In some cases, third-party tools are needed, such as innobackupex for MySQL.
2. Copy the snapshot to the new follower node.
3. The follower connects the the leader and requests all the data changes that have happened since the snapshot was taken. This requires that the snapshot is associated with an exact position in the leader's replication log. That position has various names: for example, PostgreSQL calls it the log sequence number, and MySQL calls it the binlog coordinates.
4. When the follower has processed the backlog of data changes since the snapshot, we say it has caught up. It can now continue to process data changes from the leader as they happen.  

The practical steps of setting up a follower vary significantly by database. In some systems the process is fully automated, whereas in others in can be a somewhat arcane multi-step workflow that needs to be manually performed by administrators.  

# Handling Node Outages

Any node in the system can go down, perhaps unexpectedly due to a fault, but just as likely do to planned maintenance (for example, rebooting a machine to install a kernel security patch.) Being able to reboot individual nodes without downtime is a big advantage for operations and maintenance. Thus, our goal is to keep the system as a whole running despite node failures, and to keep the impact of a node outage as small as possible.  

How do you achieve high availability with leader-based replication?  

### Folower failure: Catch-up recovery

On its local disk, each follower keeps a log of the data changes it has received from the leader. If a follower crashes and is restarted, or if the network between the leader and the followers is temporarily interrupted, the follower can recover quite easily from its log, it knows the last transaction that was processed before the fault occured. Thus, the follower can connect to the leader and request all the data changes that occured during the time when the follower was disconnected. When it has applied these changes as before.  

### Leader failure: Failover

Handling a failure of the leader is trickier: one of the followers needs to be promoted to the new leader, clients need to be reconfigured to send their writes to the new leader, and the other followers need to start consuming data changes from the new leader. This process is called failover.  

Failover can happen manually (an administrator is notified that the leader has failed and takes the necessary steps to make a new leader) or automatically. An automatic failover process usually consists of the following steps:  

1. Determining that the leader has failed. There are many things that could potentially go wrong: crashes, power outages, network issues, and more. There is no foolproof way of detecting what has gone wrong, so most systems simply use a timeout: nodes frequently bounce messages back and forth between each other, and if a node doesn't respond for some period of time -- say, 30 seconds-- it is assumed to be dead. (If the leader is deliberately taken down for planned maintenance, this doesn't apply.)

2. Choosing a new leader. This could be done through an election process (where the leader is chosen by a majority of the remaining replicas), or a new leader could be appointed by a previously elected controller node. The best candidate for leadership is usually the replica with the most up-to-date changes from the old leader (to minimize any data loss). Getting all the nodes to agree on a new leader is a consensus problem, discussed in detail in Chapter 9.

3. Reconfiguring the system to use the new leader. Clients now need to send thier write request to the new leader (we discuss this in "Request Routing" on page 214). If the old leader comes back, it might still believe that it is the leader, not realizing that the other replicas have forced it to step down. The system needs to ensure that the old leader becomes a folllower and recognizes the new leader.  

Failover is fraught with things that can go wrong:  

- If asynchronous replication is used, the new leader may not have received all the writes from the old leader before it failed. If the former leader rejoins the cluster after a new leader has been chosen, what should happen to those writes? The new leader may have received conflicting writes in the meantime. The most common solution is for the old leader's unreplicated writes to simply be discarded, which may violate clients' durability expectations.  

- Discarding writes is especially dangerous if other storage systems outside of the database need to be coordinated with the database contents. For example in one incident at GitHub, an out-of-date MySQL follower was promoted to leader. The database used an autoincrementing counter to assign primary keys to new rows, but because the new leader's counter lagged behind the old leader's, it reused some primary keys that were previously assigned by the old leader. These primary keys were also used in a Redis store, so the reuse of primary keys resulted in inconsistency between MySQL and Redis, which cause some private data to be disclosed to the wrong users.  

- In certain fault scenarios (see Chapter 8), it could happen that two nodes both beleive that they are the leader. This situation is called split brain, and it is dangerous: if both leaders accept writes, and there is no process for resolving conflicts, data is likely to be lost or corrupted. As a safety catch, some systems have a mechanism to shut down one node if two leaders are detected. However, if this mechanism is not carefully designed, you can end up with both nodes being shut down.  

- What is the right timeout before the leader is declared dead? A long timeout means a longer time to recovery in the case where the leader fails. However, if the timeout is too short, there could be unecessary failovers. For example, a temporary load spike could cause  node's response time to increase above the timeout or a network glitch could cause delayed packets. If the system is already struggling with high load or network problems, an unnecessary failover is likely to make the situation worse, not better.  

There are no easy solutions to these problems. For this reason, some operations eams prefer to perform failover manually, even if the software supports automatic failover.  

These issues -- node failures; unreliable networks; and trade-offs around replica consistency, durability, availability, and latency -- are in fact fundamental problems in distributed systems. In Chapter 8 and Chapter 9 we discuess them in greater depth.  

### Implementation of Replication logs

How does leader-based replication work under the hood? Several different replication methods are used in practice, so let's look at each one briefly.  

### Statement-based replication  

In the simplest case, the leader logs every write request (statement) that is executes and sends that statement log to its followers. For a relational database, this means that every INSERT, or DELETE statement is forwarded to followers, and each follower parse and executes that SQL statement as if it had been received from a client.  

Althought this may sound reasonable, there are various ways in which this approach to replication can break down:  

- Any statement that calls a nondeterministic function, such as NOW() to get the current date and time or RAND() to get a random number, is likely to generate different value on each replica.  

- If statements use an autoincrementing column, or if they dependo n the existing data in the database (e.g., UPDATE ... WHERE... or some other condition). they must be executed in exactly the same order on each replica, or else they may have a different effect. This can be limiting when there are multiple concurrently executing transations.  

- Statements that have side effects(e.g., triggers, stored procedures, user-defined fucntions) may result in different side effects occurring on each replica, unless the side effects are absolutely deterministic.  

It is possible to work around those issues -- for example, the leader can replace any nondeterministic function calls with a fixed return value when the statement is logged so that the followers all get the same value. However, because there are so many edge cases, other replication methods are now generally preferred.  

Statement-based replication was used in MySQL before version 5.1. It is still sometimes used today, as  it is quite compact, but by default MySQL now switches to rowbased replication if there is any nondeterminism in a statement. VoltDB uses satement-based replication, and makes it safe by requiring transactions to be deterministic.  

### Write-ahead log (WAL) shipping

In Chapter 3 we discussed how storage engines represent data on disk, and we found that usually every write is appended to a log:  

- In the case of a log-structured storage engine(see "SSTables and LSM-Trees" on page 76), this log is the main place for storage. Log segments are compacted and garbage-collected in the background.  

- In the case of a B-tree, which overwrites individual disk blocks, every modification is first written to a write-ahead log so that the index can be restored to a consistent state after a crash.  

In either case, the log is an append-only sequence of bytes containing all writes to the database. We can use the exact same log to build a replica on another node: besides writting the log to disk, the loeader also sends it across the network to its followers.  

When the follower process this log, it builds a copy of the exact same data structures as found on the leader.  

This method of replication is used in PostgreSQL and Oracle, among others. The main disadvantage is that the log describes the data on a very low level: a WAL contains details of which bytes were changed in which disk blocks. This makes replication closely coupled to the storage engine. If the database changes its storage format from one version to another, it is typically not possible to run different versions of the database software on the leader and the followers.  

That may seem like a minor implementation detail, but it can have a big operation impact. If the replication protocol allows the followers to use a newer software version than the leader, you can perform a zero-downtime upgrade of the database software by first upgrading the followers and then performing a failover to make one of the upgrades nodes the new leader. If the replication protocol does not allow this version mismatch, as often the case with WAL shipping, such upgrades require downtime.  

### Logic(row-based) log replication

An alternative is to use different log formats for replication and for the storage engine, which allows the replication log to be decoupled from the storage engine internals. This kind of replication log is called a logical log, to distinguish it from the storage engine's (physical) data representation.  

A logical log for a relational database is usually a sequence of records describing writes to database tables at the granularity of a row:  

- For an inserted row, the log contains the new values of all columns.  
- For a deleted row. the log contains enough information to uniquely identify the row that was deleted. Typically this would be the primary key, but if there is no primary key on the table, the old values of all columns need to be logged.  

- For an updated row, the log contains enough information to uniquely identify the updated row, and the new values of all columns (or at least the new values of all columns that changed).  

A transaction that modifies several rows generates several such log records. followed be a record indicating that the transaction was commited. MySQL's binlog (when configured to use row-based replication) uses this approach.  

Since a logical log is decoupled from the storage engine internals, it can more easily be kept backward compatible, allowing the leader and the follower to run different versions of the database software, or even different storage engines.  

A logical log format is also easier for external applications to parse. This aspect is useful if you want to send the contents of a database to an external system, such as a datawarehouse for offline analysis, or for building custom indexes and caches. This technique is called change data capture, and we will return to it in Chapter 11.  

### Triger-based replication

The replication approaches described so far are implemented by the database system, without involving any application code. In many cases, that's what you want --but there are some circumstances where more flexibility is needed. For example, if you want to only replicate a subset of the data, or want to replicate from one kind of database to another, or if you need conflict resolution logic , then you may need to move replication up to the application layer.  

Some tools, such as Oracle GoldenGate, can make data changes available to an application by reading the database log. An alternative is to use features that are available in many relational databases: triggers and stored procedures.  

A trigger lets you register custom application code that is automatically executed when a data changes (write transaction) occurs in a database system. The trigger has the opportunity to log this change into a separate table, from which it can read by an external process. That external process can then apply any necessary application logic and replicate the data change to another system. Databus for Oracle and Bucardo for Postgres works like this, for example.  

Triger-based replication typically has greater overheads that other replication methods, and it more prone to bugs and limitations than the databases's built-in replication. However, it can nevertheless be useful due to its flexibility.  

### Problems with replication Lag

Being able to tolerate node failures is just one reason for wanting replication. As mentioned in the introduction to Part 2, other reasons are scalability (processing more requests than a single machine can handle) and latency (placing replicas geographically closer to users).  

Leader-based replication requires all writes to go through a single node, but read-ony queries can go to any replica. For workloads that consist of mostly reads and only a small percentage of writes (a common pattern on the web), thre is an atractive option: create many followers, and distribute the read requests across those followers. This removes load from the leader and allows read request to be served by nearby replicas.  

In this read-scaling archietecture, you can increase the capacity for serving read-only requests simply by adding more followers. However, this approach only realistically works with asyncronous replication--if you tried to snchronously replicate to all followers, a single node failure or network outage would make the entire system unavailabl for writing. And the more nodes you have, the more likely it is that one will be down, so a fully synchronous configuration would be very unreliable.  

Unfortunately, if an application reads from the asynchronous follower, it may see outdated information if the folllower had fallen behind. This leads to apparent inconsistencies in the database: if you run the same query on the leader and a follower at the same time, you may get different results, because not all writes have been reflected in the followers. This inconsistency is ust a temporary state--if you stop writing to the database and wait a while, the followers will eventually catch up and become consistent with the leader. For that reason, this effect is known as eventual consistency.  

The term "evenntually" is deliberately vague: in general, there is no limit to how far a replica can fall behind. In normal operation, the delay between a write happening on the leader and being reflected on a follower -- the replicaation lag-- may be only a fraction of a second, and not noticeable in practice. However, if the system is operating near capacity or if there is a problem in the network, the lag can easily increase to serveral seconds or even minutes.  

When the lag is so large, the inconsistencies it introduces are not just a theoretical issue but a real problem for applications. In this section we will highlight three examples of problems that are likley to occur when there is replicatino lag and outline some approaches to solving them.  

### Reading Your Own Writes

Many applications let the user submit some data and then view what they have submitted. This might be a record in a customer database, or a comment on a discussion thread, or something else of that sort. When new data is submitted, it must be sent to the leader, but when the user views the data, it can be read from a follower. This is especially appropriate if data is frequently viewed but only occasionally written.  

With asynchronous replication, there is a problem, illustrated in Figure 5-3: if the user views the data shortly after making a write, the new data may not yet have reached the replica. To the user, it looks as though the data they submitted was lost, so they will be undertstandably unhappy.  

In this situation, we need read-after-write consistency, also known as read-your-writers consistency.  This is a guarantee that is the user reloads the page, they will always see any updates they submitted themselves. It makes no promises about other users: other users' updates may not be visible until some later time. However, it reassures the user that their own input has been saved correctly.  

How can we implement read-after-write consistency in a system with leader-based replication? There are various possible techniques. To mention a few:  

- When reading something that the user may have modified, read it from the leader; otherwise, read it from a follower. This requires that you have some way of knowing whether someting might have been modified, without actually querying for it. For example, user profile information on a social network is normally only editable by the owner of the profile, not by anybody else. Thus, a simple rule is: always read the user's own profile from the leader, and any other user' profiles from a follower.  

- If most things in the application are potentially editable by the user, that approach won't be effective, as most things would have to be read from leer (negating the benefit of read scaling). In that case, other criteria may be used to decide whether to read from the leader. For example, you could track the time of the last update and, for one minute after the last update, make all reads from the leader. You could also monitor the replication lag on followers and prevent queries on any follower that is more than one minute behind the leader.  

- The client can remember the timestamp of its most recent write--then the system can ensure that the replica serving any reads for that user reflects updates at least until that timestamp. If a replica is not sufficiently up to date, either the read can be handled by another replica or the query can wait until the replica has caught up. The timestamp could be a logical timestamp (something that indicates ordering of writes, such as the log sequence number) or that actual system clock (in which case clock synchronization becomes critical)  

- If your replicas are distributed across multiple datacenters (for geographical proximity to users or for availability), there is additional complexity. Any request needs to be served by the leader must be routed to the datacenter that contins the leader.  

Another complication arises when the same user is ccessing your service from multiple devices, for example a desktop web browser and a mobile app. In this case you may want to provide cross-device read-after-write consistency: if the user enters some information on one device and then views it on another device, they should see the information they just entered.  

In this case, there are some additional issues to consider:  

- Approaches that require remembering the timestamp of the user's last update become more difficult, because the code running on one device doesn't know what updates have happened on the other device. This metadata will need to be centralized.  

- If your replicas are distributed across different datacenters, there is no guarantee that connections from different devices will be routed to the dame datacenter (For example, if the user's desktop computer uses the home broadband connection and their mobile device uses the cellualar data network, the device's network routes may be completely different.) If your approach requires reading from the leader, you may first need to route requests from all of a user's devices to the same datacenter.  

### Monotonic Reads  

Our second example of an anomaly that can occur when reading from asynchronous followers is that it's possible for a user to see things moving backward in time.  

This can happen if a user makes several reads from different replicas. For example Figure 5-4 shows user 2345 making the same query twice, first to a follower with little lag, then to a follower with greater lag. (This scenrio is quite likely if the user refreshes a web page, and each request is routed to a random server.) The first query returns a comment that was recently added by user 1234, but the second querry doesn't return anything because the lagging follower has not yet picked up that writes. In effect, the second query is observing the system at an earlier point in time than the first query. This wouldn't be so bad if the first query hadn't returned anything becuase user 2345 probably wouldn't know that user 1234 had recently added a comment. However, it's very confusing for user 2345 if they first see user 1234's comment appear, and then see it disappear again.  

Monotonic reads is a guarantee that this kind of anomaly does not happen. It's a lesser guarantee than strong consistency, but a stronger guarantee than eventual consistency. When you read data, you may see an old value; monotonic reads only means that if one user makes several reads in sequence, they will not see time go backward -- i.e., they will not read older data after having previously read newer data.  

One way of achieving monotonic reads is to make sure that each user always makes their reads from the same replica different users can read from different replicas). For example, the replica can be chosen based on a hash of the user ID, rather than randomly. However, if that replica fails, the user's queries will need to be rerouted to another replica.  

### Consistent Prefix Reads

Our third example of replication lag anomalies concerns violation of causality. Imagine the following short dialog between Mr. Poons and Mrs. Cake:   

Mr. Poons  
- How far into the future can you see, Mrs. Cakes?

Mrs. Cake  
- About ten seconds usually, Mr. Poons.  

There is a causal dependency between those two sentences: Mrs. Cake heard Mr. Poon's question and answered it.  

Now, imagine a third person is listening to this conversation through followers. The things said by Mrs. Cake go through a follower with little lag, but the things said by Mr. Poons have a longer replication lag. This observer would hear the following:  

Mrs. Cake  
- About ten seconds usually, Mr. Poons

Mr. Poons  
- How far into the future can you see, Mrs. Cake?  

To the ovserver it looks as though Mrs. Cake is answering the question before Mr. Poons has even asked it. Such psychic powers are impressive, but very confusing.  

Preventing this kind of anomaly requires another tupe of guarantree: consistent prefix reads. This guarantee says that if a sequence of writes happens in a certain order, then anyone reading those writes will see them appear in the same order.  

This is a particular problem in partitioned (sharded) databases, which we will discuss in Chapter 6. If the database always applies writes in the same order, reads always see a consistent prefix, so this anomaly cannot happen. However, in many distributed databases, different partitions operate independently, so there is no global ordering of writes: when a user reads from the database, they may see some parts of the database in an older state and some in a newer state.  

One solution is to make sure that any writes that are causually related to each other are written to the same partition --but in some applications that cannot be done efficiently. There are also algorithms that explicityly keep tracj of causual dependencies, a topic that we will return to in "The "happens-before" relationship and concurrency" on page 186.  

### Solutions for Replication Lag

When working with an eventually consistent system, it is worth thinking about how the application behave if the replication lag increases to several minutes or even hours. If the answer is "no problem," that's great. However, if the result is a bad experience for users, it's important to design the system to provide a stronger guarantee, such as read-after-write. Pretending that replication is synchronous when in fact it is asynchronous is a recipe for problems down the line.  

As discussed earlier, there are ways in which an application can provide a stronger guarantee than the underlying database--for example, by performing certain kinds of reads on the leader. However, dealing with thses issues in application code is complex and easy to get wrong.  

It would be better if application developers didn't have to worry about subtle replication issues and could just trust their databases to "do the right think." This is why transactions exist: they are a way for a database to provide stronger guarantees so that the application can be simpler.  

Sngle-node transactions have existed for a long time. However, in the move to distributed (replicated and partitioned) databases, many systems have abandoned them, claiming that transactions are too expensive in terms of performance and availability, and asserting that eventual consistency is inevitable in a scalable system. There is some trusth in that statement, but it is overly simplistic, and we will develop a more nuanced view over the course of the rest of this book. We will return to the topic of transactions in Chapteers 7 and 9, and we will discuss some alternative mechanisms in Part 3.  

### Multi-Leader Replication

So far in this chapter we have only considered replication architectures using a single leader. Althought that is a common approach, there are interesting alternatives.  

Leader-based replication has one more major downside: there is only one leader, and all writes must go through it. If you can't connect to the leader for any reason, for example due to a network interruption between you and the leader, you can't write to the database.  

A natural extension of the leader-based replication model is to allow more than one node to accept writes. Replication still happens in the same way: each node that processes a write must forward that data change to all the other nodes. We call this a multi-leader configuration (also known as master-master or active/active replication). In this setup, each leader simultaneously acts as a follower to the other leaders.  

### Use Cases for Multi-Leader Replication  

It rarely makes sense to use multi-leader setup with a single datacenter, because the benefits rarely outweight the added complexity. However, there are some situations in which this configuration is reasonable.  

### Multi-datacenter operation

Image you have a database with replicas in several different datacenters (perhaps so that you can tolerate failure of an entire datacenter, or perhaps in order to be closer to your users.). With a normal leader-based replication setup, the leader has to be in one of the datacenters, and all writes must go through the datacenter.  

In a multi-leader configuration, you can have a leader in each datacenter. Figure 5-6 shows what this architecture might look like. Within each datacenter, regular leaderfollowers replication is used; between datacenters, each datacenter's leader replication its change to the leader in other datacenters.  

Let's compare how the single-leader and multi-leader configurations fare in a multi-datacenter deployment:  

Performance  
- In a single-leader configuration, every write must go over the internet to the datacenter with the leader. This can add significant latency to writes and might contravene the purpose of having multiple datacenters in the first place. In a multi-leader configuration, every write can be processed in the local datacenter and is replicated asynchronously to the other datacenters. Thus, the interdatacenter network delay is hidden from users, which means the perceived perfromance may be better.  
    
Tolernce of datacenter outages  
- In a single-leader configuration, if the datacenter with the leader fails, failover can promote  folllower in another datacenter to be leader. In a multi-leader configuration, each datacenter can continue operating independently of the others, and replication catches up when the failed datacenter comes back online.  

Tolerance of network problems  
- Traffic between datacenters usually goes over the public internet, which may be less reliable than the local network within a datacenter. A single-leader configuration is very sensitive to problems in the inter-datacenter link, because writes are made synchronously over this link. A multi-leader configuration with asynchronous replication can usually tolerate network problems better: a temporary network interruption does no prevent writes being processed.  

Some database support multi-leader configurations by default, but it is also often implemented with external tools, such as Tungsten Replicator for MySQL, for PostgreSQL, and GoldenGate for Oracle.  

Althought multi-leader replication has advantages, it also has a big downside: the same data may be concurrently modified in two different datacenters, and those write conflicts must be resolved (indicated as "conflict resolution" in Figure 5-6). We will discuss this issue in "Handling write conflicts" on page 171.  

As multi-leader replication is a somewhat retrofitted feature in many databases, there are often subtle configuration pitfalls and surprising interactions with other database features. For example, autoincrementing keys, triggers, and integrity constraints can be problematic. For this reason, multi-leader replication is often considred dangerous territory that should be avoided if possible.  

### Clients with offline operations

Another situation in which multi-leader replication is appropriate is if you have an application that needs to continue to work while it is disconnected from the internet.  

For example, consider the calendar apps on your mobile phone, your laptop, and other devices. You need to be able to see your meetings (make read requests) and enter new meetings (make write requests) at any time, regardless of whether your device currently has an internet connection. If you make any changes while you are offline, they need to be synced with a server and your other devices when the device is next online.  

In this case, every device has a local database that acts as a leader (it accepts write requests), and there is an asynchronous multi-leader replication process (sync) between the replicas of yor calendar on all your devices. The replication lag may be hours or even days, depending on when you have internet access available.  

From an architectural point of view, this setup is essentially the same as multi-leader replication between datacenters, taken to the extreme: each device is a "datacenter", and the network connection between them is extremely unreliable. As the rich history of broken calendar sync implementations demonstrates, multi-leader replication is a tricky thing to get right.  

There are tools that aim to make this kind of multi-leader configuration easier. For example, CouchDB is designed for this mode of operation.  

### Collaborative editing

Real-time collaborative editing applications allow several people to edit a document simultaneously. For example, Etherpad and Google Docs allow multiple people to concurrently edit a text document or spreadsheet (the algorithm is briefly discussed in Automatic conflct Resolution" on page 174).  

We don't usually think of collaborative editing as a database replication problem, but it has a lot in common with the previously mentioned offline editing use case. When one user edits a document, the changes are instantly applied to their local replica (the state of the document in their web browser or client application) and asynchronously replicated to the server and any other users who are editing the same document.  

If you want to guarantee that there will be no editing conflicts, the application must obtain a lock on the document before a user can edit. If another user wants to edit the same document, they first have to wait until the first user has commited their changes and released the lock. This collaboration model is equvalent to single-leader replication with transactions on the leader.  

However, for faster collaboration, you may want to make the unit of change very small (e.g., a single keystroke) and avoid locking. This approach allows multiple users to edit simultaneously, but it also brings all the challenges of multi-leader replication, including requiring conflict resolution.  

### Handling Write Conflicts

The biggest problem with multi-leader replication is that write conflicts can occu, which means that conflict resolution is required.  

For example, consider a wiki page that is simulaneously being edited by two users, as shown in Figure 5-7. User 1 changes the title of the page from A to B, and user 2 changes the title from A to C at the same time. Each user's change is successfully applied to their local leader. However, when the changes are asynchronously replicated, a conflict is detected. This problem does not occur in a single-leader database.  


### Synchronous versus asynchronous conflict detection

In a single-leader database, the second writer will either block and wait for the first wirte to complete, or abort the second write transaction, forcing the user to retry the write. On the other hand, in a multi-leader setup, both writes are successful, and the conflict is only detected asynchronously at some later point in time. At that time, it may be too late to ask the user to resolve the conflict.  

In principle, you could make the conflict detection synchronous --i.e., wait for the write to be replicated to all replicas before telling the user that the write was successful. However, by doing so, you would lose the main advantage of multi-leader replication: allowing each replica to accept writes independetly. If you want synchronous conflict detection, you might as well just use single-leader replication.  

### Conflict avoidance  

The simplest strategy for dealing with conflicts is to avoid them: if the application can ensure that all writes for a particular record go through the same leader, then conflicts cannot occur. Since many implementations of multi-leader replication handle conflicts quite poorly, avoiding conflicts is a frequently recommended approach.  

For example, in an application where a user can edit their own data, you can ensure that requests from a particular user are always routed to the same datacenter and use the leader in that datacenter for reading and writing. Different users may have different "home" datacenters (perhaps picked based on geographic proximity to the user), but from anyone user's point of view the coniguration is essentiallu single-leader.  

However, sometimes you might want to change the designated leader for a record--perhaps because one datacenter has failed and you need to reroute traffic to another datacenter, or perhaps because a user has movd to a different location and is now closer to a different datacenter. In this situation, conflict voidance breaks down, and you have to deal with the possiblity of concurrent writes on different leaders.  

### Converging toward a consistent state

A single-leader database applies writes in a sequential order: if there are several updates to the same field, the last write determines the final value of the field.  

In a multi-leader configuration, there is no defined ordering of writes, so it's not clear what the final value should be. In Figure 5-7, at leader 1 the title is first updated to B and then to C; at leader 2 it is first updated to C and then to B. Neither order is "more correct" than the other.  

If each replica simply applied writes in the order that it saw the writes, the database would end up in and inconsitent state: the final value would be C at leader 1 and B at leader 2. That is not acceptable --every replication scheme must ensure that the data is eventually the same in all repllicas. Thus, the database must resolve the onflict in a convergent way, which means that all replicas must arrive at the same final value when all changes have been replicated.  

There are various ways of achieving convergent conflict resolutions:  

- Give each write a unique ID (e.g., a timestamp, a long random number, a UUID, or a hash of the key and value), pick the write with the highest ID as the winner, and throw away the other writes. If a timestamp is used, this technique is known as last write wins (LWWW). Although this approach is popular, it is dangerously prone to data loss. We will discuss LWW in more details at the end of this chapter.  

- Give each replica a unique ID, and let writes that originated at a higher-numbered replica always take precedence over writes that originatd at a lower-numvered replica. This approach also implies data loss.  

- Somehow merge the values together--e.g., order them alphabetically and then concatenate them (in Figure 5-7, the merged title might be something like "B/C").  

- Record the conflict in an explicit data structure that preserves all information, and write application code that resolves the conflict at some later time (perhaps by prompting the user).  

### Customer conflict resolution logic  

As the most appropriate way of resolving a conflict may depend on the application, most multi-leader replication tools let you write conflict resolution logic using application code. That code may be executed on write or on read:  

On write:  
    - As soon as the database system detects a conflict in the log of replicated changes, it calls the confluct handler. For example, Bycardo allows you to write a snippet of Perl for this purpose. This handler typically cannot prompt a user -- it runs in a backgroun process and it must execute quickly.  
      
On Read:  
    - When a conflict is detected, all the conflicting writes are stored. The next time the data is read, these multiple versions of the data are returned to the application. The application may prompt the user or automatically resolve the conflit, and write the result back to the database. CouchDB works this way for, for example.  
    
Note that conflict resolution usally applies at the level of an individual row or document, not for an entire transaction. Thus , if you have a transaction that atomically makes several different writes, each write is still considered separately for the purpose of conflict resolution.  

### Automatic conflict resolution

Conflict resolution rules can quickly become complicated, and custom cose can be error-prone. Amazon is frequently cited example of surprising effects due to a conflict resolution handler: for some time, the confluct resoluation logic on the shopping cart would preserve items added to the cart, but not items removed from the cart. Thus, customers would sometimes see items reappearing in their carts even though they had previously been removed.  

There has been some interesting research into automatically resolving conflicts caused by concurrent data modification. A few lines of research are worth mentioning:  

- Conflict-free replicated datatypes are family of data structures for sets, maps, ordered lists , counters, etc. that can concurrentlu edited by multiple users, and which automatically resolve conflicts in sensible ways. Some have been implemented in Riak
- Mergeabke persistent data structures
- operational transformations

Implementations of these algorithms in databases are still young, but it's likely that they will integrated into more replicated data systems in the future. Automatic conflict resolution could make multi-leader data synchronization much simpler for applications to deal with.  

### What is conflict?

Some kinds of conflicts are obvious. In example in Figure 5-7, two writes concurently modified the same field in the same record, setting it to two different values. There is little douby that this is a conflict.  

Other kinds of conflict can be more subtle to detect. For example, consider a meeting room booking system: it tracks which room is booked by which group of people at which time. This application needs to ensure that each room is only booked by one group of people at any one time (i.e., there must not be any overlapping bookings for the same room). In this case, a conflict may arise if two different bookings are created for the same room at the same time. Even if the application checks availability before allowing a user to make a booking, there can be a conflict if the two bookings are made on two different leaders.  

There isn't a quick read-made answer, but in the following chapters we will trace path toward a good understanding of this problem. We will see some more examples of conflicts in chapters 7, and chapter 12 we will discuss scalable approaches for detecting and resolving conflicts in a replicated system.  

### Multi-leader Replication Topologies

A replication topology describes the communcation paths along which writes are propagated from one node to another. If you have two leaders, like in Figure 5-7, there is only one plausible topology: keader 1 must send all of its writes to leader 2, and vice versa. With more than two leaders, various different topologies are possible.  

The most general topology is all-to-all, in which every leader sends its writes to every other leader. However, more restricted topologies are also used: for example, MySQL by default supports only a circular topology, in which each node receives writes from one node and forwards those writes (plus any writes of its own) to one other node. Another popular topology has the shape of a start. One designated root node forwards writes to all of the other nodes. The star topology can be generalized to a tree.  

In circular and star topologies, a write may need to pass through several nodes before it reaches all replicas. Therefore, nodes need to forward data changes they received from other nodes. To prevent infinite replication loops, each node is given a unique identifier, and in the replication log, each write is tagged with the identifiers of all the nodes it has passed through. When a node receives a data change that is tagged with its own identifier, that data cange is ignored, because the node knows that it has already been processed.  

A problem with circular and star topologies is that if just one node fails, it can interrupt the flow of replication messages between other nodes, causing them to be unable to communicate until the node is fixed. The topology could be reconfigured to work around the failed node, but in most deployments such reconfiguration would have to be done manually. The fault tolerance of a more desnely connected topology (such as all-to-all) is better because it allows messages to travel along different paths, avoiding a single point of failure.  

On the other hand, all-to-all topologies can have issues too. In particular, some network links may be faster than others (e.g., due to network congestion), with the result that some replication messages may "overtake" others, as illustrated in Figure 5-9.  

In Figure 5-9, client A inserts a row into a table on leader 1, and client B updates that row on leader 3. However, leader 2 may reeive the writes in different order: it may first recive the update (which from its point of view, is an update to a row that does not exist in the database) and only later receive the corresponding insert (which should have preceded the update).  

This is a problem of causality, similar to the one we saw in "Consistent Prefix Reads" on page 165: the update depends on the prior insert, so we need to make sure that all nodes process the insert first, and tehn the update. Simply attaching a timestamp to every write is not sufficient, because clcks cannot be trusted to be sufficiently in sync to correctly order these events at leader 2.  

To order these events correctly, a technique called version vectors can be used, which we will discuss later in this chapter. However, conflict detection techniques are poorly implemented in many multi-leader relication systems. For example, at the time of writing, postgreSQL, BDR does not provide causal order of writes, and Tungsten Replicator for MySQL doesn't even try to detect conflicts.  

If you are using a system with multi-leader replication, it is worth being aware of these issues, carefully reading the documentation, and thoroughly testing your database to ensure that it really does provide the guarantees you beleive it to have.  

### Leaderless Replication  


























