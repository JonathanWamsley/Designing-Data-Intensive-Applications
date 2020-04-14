# Part 2: Distributed Data

In part 1 of this book, we discussed aspects of data systems that apply when data is stored on a single machine. Now, in part 2, we move up a level and ask: what happens if multiple machines are involved in storage and retrieval of data?  

There are various reasons why you might want to distribute a database across multiple machines:  

Scalability:  
- If your data volume, read load, or write load grows bigger than a single machine can handle, you can potentially spread the load scross multiple machines.  

Fault tolerance/high availability:  
- If your application needs to continue working even if one machine (or several machines, or the network, or an entire datacenter) goes down, you can use multiple machines to give you redundancy. When one fails, another one can take over.  

Latency:  
- If you have users around the world, you might want to have servers at various locations worldwide so that each user can be served from a datacenter that is geographically close to them. That avoids the users having to wait for network packets to travel half around the world.  

# Scaling to Higher Load

If all you need is to scale to higher load, the simplest approach is to buy more powerful machines (sometimes called vertical scaling or scaling up). Many CPUs, many 
RAM chips, and many disks can be joined together under one operating system, and a fast interconnect allows any CPU to access any part of the memory or disk. In this kind of shared-memory architecture, all the components can be treated as a single machine.  

The problem with a shared-memory approach is that the cost grows faster than linearly: a machine with twice as many CPUs, twice as much RAM, and twice as much disk capacity as another typically cost significantly more than twice as much. And due to bottlenecks, a machine twice the size cannot neccessarily handle twice the load.  

A shared-memory architecture may offer limited fault tolerance - high-end machines have hot-swappable components (you can replace disks, memory modules, and even CPUs without shutting down the machine) - but its definitely limited to a single geographical location.  

Another approach is the shared-disk architecture, which uses several machines with independent CPUs and RAM, but stores data on an array of disks that is shared between the machines, which are connected via a fast network. This architecture is used for some data warehousing workloads, but contention and the overhead of locking limit the scalability of the shared-disk approach.  

# Shared-Nothing Architectures 

By contrast, shared-nothing architectures (sometimes called horizontal scaling or scaling out) have gained a lot of popularity. In this approach, each machine or virtual machine running the database software is called a node. Each node uses its CPUs, RAM, and disks independently. Any coordination between node is done at the software level, using a conventional network.  

No special hardware is required by a shared-nothing system, so you can use whatever machines have the best price/performance ratio. You can potentially distribute data across multiple geographic regions, and thus reduce latency for users and potentially be able to survive the loss of an entire datacenter. With cloud deployment of virtual machines, you don't need to be operating at Google scale: even for small companies, a multi-region distributed architecture is now feasible.  

In this part of the book, we focus on shared-nothing architecture --not because they are necessarily the best choice for every use case, but rather because they require the most caution from you, the application developer. If yout data is distributed across multiple nodes, you need to be aware of the constraints and trade-offs that occur in such in such distributed systems - the database cannot magically hide these from you.  

While a distributed shared-nothing architecture has many advantages, it usually also incurs additional complexity for applications and sometimes limits the expressiveness of the data models you can use. In some cases, a simple single-threaded program can perform significantly better than a cluster with over 100 CPU cores. On the other hand. shared-nothing systems can be very powerful. The next few chapters go into details on the issues that arise when data is distributed.  

# Replication Versus Partitioning

There are two common ways data is distributed across multiple nodes:  

Replication:  
- Keeping a copy of the same data on several different nodes, potentially in different locations. Replication provides redundancy: if some nodes are unavailable, the data can still be served from the remaining nodes. Replication can also help improve performance. We discuss replication in Chapter 5.  

Partitioning:  
- Solitting a big database into smaller subsets called partitions so that different partitions can be assigned to different nodes (also known as sharding). We discuss partitioning in Chapter 6.  

He shows a file structure split into partitions by some numerical metric. Then all the distinct partitions are replicated, having multiple copies as backup.  

With an understanding of those concepts, we can discuss the difficult trade-offs that you need to make in a distributed system. We'll discuss transactions in Chapter 7, as that will help you understand all the manythings that can go wrong in a data system, and what you can do about them. We'll conclude this part of the book by discussing the fundamental limitations of distributed systems in Chapter 8 and 9.  

Later, in part 3 of this book, we will discuss how you can take several (potentialy distributed) datastores and integrate themm into a larger system, satisfying the needs of a complex application. But first, let's talk about distributed data.  