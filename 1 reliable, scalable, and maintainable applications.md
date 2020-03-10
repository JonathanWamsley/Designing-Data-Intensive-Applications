# Reliable, Scalable, and Maintainable Applications

Applications are **Data-intensive** as opposed to **Compute-intensive**. This means CPU is not the bottleneck, but the amount of data, complexity of the data and the speed at which it is changing is the bottleneck.

Common elements in data-intensive applications
- store data so that they or another application can find it again(database)
- remember the results of an expensive operation, to speed up reads(caches)
- allow users to search data by keyword or filter it in various ways (search indexes)
- send a message to another process, to handle asynchronously (stream processing)
- periodically crunch a large amount of accumulated data (batch processing)

When building applications, we need to figure out which tools and which approaches are the most appropriate for the task at hand.  
- There are many database systems with different characteristics, because different applications have different requirements.
- likewise, there are various approaces to caching, serveral ways of building serach indexes and so on.

### Thinking about data systems

Data system tools like databases, caches, queues are becoming mixtures of tools. They however all store data for sometime. Often, a single tool can do a task which means different tools are stithed together in applications resulting in a data system. 

This book focuses on 3 concepts when implementing a data system.  

**Reliability**: The system should continue to work *corretly*(perform the correct function at the desired level of performance) even in the face of *adversity* (hardware or software faults, and human error)  

**Scalability**: As the system *grows* (in data volumne, traffic volume, or complexity), there should be reasonable ways of dealing with that growth  

**Maintainability**: Over time, many different people will work on the system(engineering and operations, both maintaining current behavior and adapting the system to new use cases), and the should all be able to work *productively*

## Reliability 

intution tells us reliability means:  
- application performs the function that the user expected
- it can tolerate the user making mistakes or using the software in unexpected ways
- it performances is good enough for the required use case, under the expected load and data volume
- the system prevents any unauthorized access and abuse

Reliability - continuing to work correctly, even when things go wrong  

Faults - things that can go wrong  
Fault-tolerant / resilent - anticipating faults and can cope with them. (this can not always be done nor easily)  

Fault: defined as one component of the system deviating from its spec  
Failure: when the system as a whole stops providing the required service to the user  

Having a 0 fault is impossible, therefore designing fault-tolerance mechanisms that prevent faults from causing failures is the goal.  

To find faults, its good to triger them deliberately to find critical bugs from error handling. By deliberately inducing faults, you ensure that the fault-tolerant machinery is continually exercised and tested, which increases your confidence that faults will be handled correctly.  

Although we prefer tolerating faults over preventing faults, sometimes prevention is better like with security matters that can not be undone.

#### Hardware Faults

Until recently, redundanc of hardware components was sufficient for most applications, since it makes total failure of a single machine faily rare. As long as you can resotre a backup onto a new machine fairly quickly, the downtime in case of a failure is not catastrophic in most applications.  

However, as data volumes and applications begun using larger number of machines, which proportionally increase the rate of hardware faults. Moreover, in some cloud platforms such as AWS, it is faily common for virtual machine instances to become unabilable without warning, as the platforms are designed to prioritize flexibility and elasticity over single-machine reliability  

Hence the move towards systems that can tolerate the loss of entire machines, by using software fault-tolerance techniques in addition to hardware redundancy. These systems have operational advantages: a system can tolerate machine failures can be patched one node at a time, without downtime of the entire system. (rolling upgrade)  

#### Software Errors

Software faults often are software bugs that lie dormant for a long time until they are triggered by an unusual set of circumstances. They are often cause because of assumptions are being made about its enviorment which is true until the enviorment changes changes. Some common bugs are:

- Software bug that causes every instance of an application to crash when given a particular bad input
- A runaway process that uses up some shared resource -CPU time, memory, disk space, or network bandwidth
- A service that the system depends on that slows down, becomes unresponsive, or starts returning corrupted responses
- Cascading failures, where a small fault in one component triggers a fault in another component, which triggers more faults

There is no quick solution to software bugs, but it helps to carefully thinking about assumptions and interactions in the system; thorough testing; processing isolation; allowing processes to crash and restart; measuring, monitoring, and analyzing system behavior in a product. Tests like if a system expects to provide some guarantee, it can constantly check itself while it is running and raise an alert if a discrepancy is found.(this is data testing)

#### Human Error

The best way to reduce human errors is to combine several approaches:

- Design systems in a way that minimizes opportunities for error. (designing a user friendly API with well-designed abstractions that encourage them to do the right thing and not being too restrictive)
- Decouple the places where people make the most mistakes from the places where they can cause failures. (provide sandbox enviormentments where people can explore and experiment safely without affecting real users)
- Test thoroughly at all levels, from unit tests to whole-system integration tests and manual tests. Automated testing is widely used, well understood, and especially valuable for covering corner cases that rarely arise in normal operation
- Allow quick and easy recovery from human errors, to minimize the impact in the case of a failure. (easy to roll back configuration changes, roll out new code gradually, and provid tools to recompute data (in case old computation was incorrect))
- Set up detailed and clear monitoring, such as performance meterics and error rates. In other engineering disciplines this is referred to as telemetry(tracking movement of the data to track what is happening and for understanding failures). Monitoring can show us early warning signals and allow us to check whether any assumptions or constraints are being violated. When a problem occurs, meterics can be invaluable in diagnosing the issue.
- implement good managment practices and training - complex and important aspect

#### How important is Reliability?

Even in 'noncritical' applications we have a responsibility to our users. Reliability is important at every stage of development. Bugs in business applications cause lost productivity ( and legal risks if figures are reported incorrectly), and outages of ecommerce sites can have huge costs in terms of lost revenue and damage to reputation.  

There are situations where we choose to sacrifice reliability in order to reduce development cost(prototype product for an unproven market) or operational cost (for a service with a very narrow profit margin) - but we should be very conscious of when we are cutting corners.

#### Scalability

Scalability is the term to describe a system's ability to cope with increased load. So when a system grows from 10,000 concurrent users to 100,000 concurrent users for example. Discussing scalability means considering questions like 'If the system grows in a particular way, what are our options for coping with the growth?' and ' How an we add computing resources to handle the additional load?'

#### Describing Load

Load can be described as a few numbers which we call load parameters. The best choice of parameters depeond on the architecture of your system:  
- requests per second to a web server
- the ratio of reads to writes in a database
- the number of simultaneously active users in a chat room
- the hit rate on a cache
- or something else

#### Describing Performance

Once you have described the load on your system, you can investigate what happens when the load increases by loking at:  
- When you increase a load parameter and keep the system resources (CPU, memory, network bandwidth, etc) unchanged, how is the performance of your system affected?
- When you increase a load parameter, how much do you ened to increase the resources if you want to keep the performance unchanged?

Both questions requre performance numbers.  

In batch processing system like Hadoop, the number of records we can process per second, or the total time it takes to run a job on a dataset of a certain size is measured
In online systems, the service's response, that is the time between a client sending a request and receiving a response is measured by looking at the distribution since the time can vary per query. It is useful to measure the median, mean(arithmetic), 95th and 99th percentile when looking at requests. Mean is a good measurement to for the typical response. Percentiles are useful if you sort them by time and then take the median point, which means half take n ms to complete, which is good to see how long a user typical user has to wait. The percentiles are important because they impact users' experence of the service.

'Amazon has also observed that a 100ms increase in response time reduces sales by 1%, and others report that a 1-second slowdown reduces a customers satisfaction metric by 16%'  

On the otherhand, the 99th percentile may be too expensive to improve and not yield enough benifits.


