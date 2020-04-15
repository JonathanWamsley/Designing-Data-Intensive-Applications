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

































