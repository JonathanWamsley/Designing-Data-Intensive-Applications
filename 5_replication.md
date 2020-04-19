# Replication

Replication means keeping a copy of the same data on multiple machines that are connected via a network. As discussed in the introduction in Part 2, there are several reasons why you might want to replicate data:  

- To keep data geographically close to your users (and thus reduce latency)
- To allow a system to continue working even if some of its parts have failed (and thus increase availability)
- To scale out the number of machines that can server read queries (and thus increase read throughput)

In this chapter we will assume that your dataset is so small that each machine can hold a copy of the entire dataset. In Chapter 6 we will relax that assumption and discuss partitioning (sharing) of datasets that are too big for a single machine. In later chapters we will discuss various kinds of faults that can occur in a replicated data system, and how to deal with them. 

If the data you're replicaing does not change over time, then replicating is easy: you just need to copy the data to every node once, and you're done. All of the difficulty in replication lies in handling changes to replicated data, and that's what this chapter is about. We will discuss three popular algorithms for replicating changes between nodes: single-leader, multi-leader, and leaderless replication. Almost all distributed databases use one of these three approaches. They all have various pros and cons, which we will examine in detail.  

There are many trade-offs to consider with replication: for example, wheter to use synchronous or asynchronous replication, and how to handle failed replicas. Those are often configuration options in databases, and although the details vary by database, the general principles are similar across many different implementations. We will discuss the consequences of such choices in the chapter.  

Replication of databases is an old topic -- the principles haven'y changed much since they were studied in the 1970s, because the fundamental constants of networks have remained the same. Howeverk outside of research, many developers continued to assume for a long time that adatabase consisted of just one node. Minstream use of distributed databases is more recent. Since many application developers are new to the area, there has been a lot of misunderstanding around issues such as eventual consistency. In "Problems with Replication Lag" on page 161 we will get more precise about event consistency and discuss things like the read-your=writes and monotonic reads guarantees.  

# Leaders and Followers  

Each node that stories a copy of the datbase is called a replica. With mulitple replicas, a question inevitably arises: how do we ensure that all the data ends up on all the replicas?  

Every write to the database needs to be process by every replica; otherwise, the replicas would no longer contain the same data. The most common solution for this is called leader-based replication (also known as active/passive of master-slave replication) and is illustrated in Figure 5-1. It works as follows:  

1. One of the replicas is designated the leader (also known as master or primary). When clients want to write to the database, they must send their requests to the leader, which first writes the new data to its local storage.  

2. The other replicas are known as followers (read replicas, slaves, secondaries, or hot standbys). Whenever the leader writes new data to its local storage, it also sends the data change to all of its followers as part of replication log or change stream. Each follower takes the log from the leader and updates its local copty of the database accordingly, by applying all the writes in the same order as they were processed on the leader.  

3. When a client wants to read from the database, it can query either the leader or any of the followers. However, writes are only accepted on the leader (the followers are read-only from the client's point of view).  

This mode of replication is built-in feature of many relational databases, such as PostgreSQL, MySQL, Oracle Data Gyard, and SQL Server's AlwaysOn Availability Groups. It is also used in some nonrelational databases, including MongoDB, RethinkDB, and Espresso. Finally, leader-based replication is not restricted to only databases: distributed message brokers such as Kafka and RabbitMQ highly abailable queues also use it. Some network filesystems and replicated block devices such as DRBD are similar.  
# Synchronous Versus Asynchronous Replication

An important detail of replicated systems is whether the replication happens synchronously or asynchronously. (in relational databases, this is often a configurable option; other systems are often hardcoded to be either one or the other.)  

Think about what hapens in Figure 5-1, where the user of a website updates their profile image. At some point in time, the client sends the update request to the leader; shortly afterward, it is received by the leader. At some point, the leader forwards the data change to the followers. Eventually, the leader notifies the client that the update was succesful.  

Figure 5-2 shows the communication between various components of the system: the user's client, the leader, and two followers. Time flows from left to right. A request or response message is shown as a thick arrow.  

In the example of Figure 5-2, the replication to follower 1 is synchronous: the leader waits until follower 1 has confirmed that it received the write before reporting success to the user, and before making the write visible to other client. The replication to follower 2 is asychronous: the leader sends the message, but doesn't wait for a response from the follower.  

The diagram shows that there is substantial delay before followers 2 processes the message. Normally, replication is quite fast: most database system apply changes to followers in less than a second. However, there is no guarantee of how long it might take. There are circumstances when followers might fall behind the leader by serveral minutes or more; for example, if a follower is recovering from a failure, if the system is operating near maximum capacity, or if there are network problems between the nodes.  

The advantage of synchronous replication is that the follower is guaranteed to have an up-to-date copy of the data that is consisten with the leader. If the leader suddenly fails, we can be sure that the data is still available on the follower. The disavantage is that if the synchronous follower doesn't respond (because it has crashed, or there is a network fault, or for any other reason), the write cannot be processed. The leader must block all writes and wait until the synchronous replica is available again.  

For that reason, it is impractical for all followers to be synchronous: any one node outage would cause the whole system to grind to a halt. In practice, if you enable synchronous replication on a database, it usually means that one of the followers is synchronous, and the others are asynchronous. If the synchronous follower becomes unavailable or slow, one of the asynchronous followers is made synchronous. This guarantees that you have an up-to-date copy of the data on at least two nodes: the leader and one synchronous follower. The configuration is sometimes called semi-synchronous.  

Often, leader-based replication is configured to be completely asynchronous. In this case, if the leader fails and is not recovereable, any writes that have not yet been replicated to followers are lost. This means that a write is not guaranteed to be durable, even if it has been confirmed to the client. However, a fully asynchronous configuration has the advantage that the leader can continue processing writes, even if all of its followers have fallen behind.  

Weakening durability may sound like a bad trade-off, but asynchronous replication is neverthelss widely used, especially if there are many followers or uf they are geographically distributed. We will return to this issue in "Problems with Replication Lag" on page 161.  

# Research on Replication

It can be a serious problem for asynchronous replicated systems to lose data if the leader fails, so researchers have continued investigating replication methods that do not lose data but still provide good performane and availability. For example, chain replication is a variant of synchronous replication that has been successfully implemented in a few systems such as Microsoft Azure Storage.  

There is strong connection between consistency of replication and consensus (getting several nodes to agree on a value), and we will explore this area of theory in more detil in Chapter 9. In this chapter we will concentrate on the simpler forms of replication that are most commonly used in databases in practice.  

# Setting Up New Followers 

From time to time, you need to set up new followers -- perhaps to increase the number of replicas, or to replace failed nodes. How do you ensure that the new followers has an accurate copy of the leader's data?  

Simply copying data files from one nodes to another is typically not sufficient: client are constantly writting to the database, and the data is always in flux, so a standard file copy would see different parts of the database at different points in time. The result might not make any sense.  

You could make the file on disk consisten by locking the database (making it unabailable for writes), but that would go against our goal of high availability. Fortunately, setting up a follower can usually be done without downtime. Conceptually, the process looks like this:  

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

3. Reconfiguring the system to use the new leader. Clients now need to send thier write reqyest to the new leader (we discuss this in "Request Routing" on page 214). If the old leader comes back, it might still believe that it is the leader, not realizing that the other replicas have forced it to step down. The system needs to ensure that the old leader becomes a folllower and recognizes the new leader.  

Failover is fraught with things that can go wrong:  

- If asynchronous replication is used, the new leader may not have received all the writes from the old leader before it failed. If the former leader rejoins the cluster after a new leader has been chosen, what hsould happen to those writes? The new leader may have received conflicting writes in the meantime. The most common solution is for the old leader's unreplicated writes to simply be discarded, which may violate clients' durability expectations.  

- Discarding writes is especially dangerous if other storage systems outside of the database need to be coordinated with the database contents. For example in one incident at GitHub, an out-of-date MySQL follower was promoted to leader. The database used an autoincrementing counter to assign primary keys to new rows, but because the new leader's counter lagged behind the old leader's, it reused some primary keys that were previously assigned by the old leader. These primary keys were also used in a Redis store, so the reuse of primary keys resulted in inconsistency between MySQL and Redis, which cause some private data to be disclosed to the wrong users.  

- In certain fault scenarios (see Chapter 8), it could happen that two nodes both beleive that they are the leader. This situation is called split brain, and it is dangerous: if both leaders accept writes, and there is no process for resolving conflicts, data is likely to be lost or corrupted. As a safety catch, some systems have a mechanism to shut down one node if two leaders are detected. However, if this mechanism is not carefully designed, you can end up with both nodes being shut down.  

- What is the right timeout before the leader is declared dead? A long timeout means a longer time to recovery in the case where the leader fails. However, if the timeout is too short, there could be unecessary failovers. For example, a temporary load spike could cause  node's response time to increase above the timeout or a network glitch could cause delayed packets. If the system is already struggling with high load or network problems, an unnecessary failover is likely to make the situation worse, not better.  

There are no easy solutions to these problems. For this reason, some operations eams prefer to perform failover manually, even if the software supports automatic failover.  

These issues -- node failures; unreliable networks; and trade-offs around replica consistency, durability, availability, and latency -- are in fact fundamental problems in distributed systems. In Chapter 8 and Chapter 9 we discuess them in greater depth.  

### Implementation of Replication logs

How does leader-based replication work under the hood? Several different replication methods are used in practice, so let's look at each one briefly.  

### Statement-based replication  

In the simplest case, the leader logs every write request (statement) that is executes and sens that statemetn log to its followers. For a relational database, this means that every INSERT, or DELETE statement is forwarded to followers, and each follower parse and executes that SQL statement as if it had been received from a client.  

Althought this may sound reasonable, there are various ways in which this approach to replication can break down:  

- Any statement that calls a nondeterministic function, such as NOW() to get the current date and time or RAND() to get a random number, is likely to generate different value on each replica.  

- If statements use an autoincrementing column, or if they dependo n the existing data in the database (e.g., UPDATE ... WHERE... or some other condition). they myst be executed in exactly the same order on each replica, or else they mayhave a different effect. This can be limiting when there are multiple concurrently executing transations.  

- Statements that have side effects(e.g., triggers, stored procedures, user-defined fucntions) may result in different side effects occurring on each replica, unless the side effects are absolutely deterministic.  

It is possible to work around those issues -- for example, the leader can replace any nondeterministic function calls with a fixed redturn value when the statement is logged so that the followers all get the same value. However, because there are so many edge cases, other replication methods are now generally preferred.  

Statement-based replication was used in MySQL before version 5.1. It is still sometimes used today, as  it is quite compact, but by default MySQL now switches to rowbased replication if there is any nondeterminism in a statement. VoltDB uses satement-based replication, and makes it safe by requiring transactions to be deterministic.  

### Write-ahead log (WAL) shipping

In Chapter 3 we discussed how storage engines represent data on disk, and we found that usually every write is appended to a log:  

- In the case of a log-structured storage engine(see "SSTables and LSM-Trees" on page 76), this log is the main place for storage. Log segments are compacted and garbage-collected in the background.  

- In the case of a B-tree, which overwrites individual disk blocks, every modification is first written to a write-ahead log so that the index can be restored to a consistent state after a crash.  

In either case, the log is an append-only sequence of bytes containing alll writes to the database. We can use the exact same log to build a replica on another node: besides writting the log to disk, the loeader also sends it across the network to its followers.  

When the follower process this log, it builds a copy of the exact same data structures as found on the leader.  

This method of replication is used in PostgreSQL and Oracle, among others. The main disadvantage is that the log describes the data on a very low level: a WAL contains details of which bytes were changed in which disk blocks. This makes replication closely coupled to the storage engine. If the database changes its storage format from one version to another, it is typically not possible to run different versions of the database software on the leader and the followers.  

That may seem like a minor implementation detail, but it can have a big operation impact. If the replication protocol allows the followers to use a newer software version than the leader, you can perform a xzero-downtime upgrade of the database software by first upgrading the followers and then performing a failover to make one of the upgrades nodes the new leader. If the replication protocol does not allow this version mismatch, as often the case with WAL shipping, such upgrades require downtime.  

### Logic(row-based) log replication

An alternative is to use different log formats for replication and for the storage engine, which allows the replication log to be decoupled from te storage engine internals. This kind of replication log is called a logical log, to distinguish it from the storage engine's (physical) data representation.  

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