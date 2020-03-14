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

However now-a-days, data volumes and applications are using larger number of machines, which proportionally increase the rate of hardware faults. Moreover, in some cloud platforms such as AWS, it is faily common for virtual machine instances to become unabilable without warning, as the platforms are designed to prioritize flexibility and elasticity over single-machine reliability  

Hence the move towards systems that can tolerate the loss of entire machines, by using software fault-tolerance techniques in addition to hardware redundancy. These systems have operational advantages: a system can tolerate machine failures can be patched one node at a time, without downtime of the entire system. (rolling upgrade). 

AWS manages a lot of the hardware faults for its users and often provide guarantees in very high percentiles.

#### Software Errors

Software faults often are software bugs that lie dormant for a long time until they are triggered by an unusual set of circumstances. They are often cause because of assumptions are being made about its enviorment which is true until the enviorment changes. Some common bugs are:

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

Scalability is the term to describe a system's ability to cope with increased load. So when a system grows from 10,000 concurrent users to 100,000 concurrent users for example. Discussing scalability means considering questions like:  
- If the system grows in a particular way, what are our options for coping with the growth?  
- How an we add computing resources to handle the additional load?  

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

In batch processing system like Hadoop measure:
- Througput: the number of records we can process per second
- Response time: the time between a client sending a request and receiving a response
- Total time: how long it takes to run a job on a dataset of a certain size

In online systems, the service's response, that is the time between a client sending a request and receiving a response is measured by looking at the distribution since the time can vary per query. It is useful to measure the median, mean(arithmetic), 95th and 99th percentile when looking at requests.  
- Mean is a good measurement to for the typical response
- Percentiles are useful if you sort them by time and then take the middle point the median  
- Median, which means half take it takes 'X' ms to complete, which is good to see how long a user typical user has to wait
- The percentiles are important because the high levels show how a users may experence the service.  

- 'Amazon has also observed that a 100ms increase in response time reduces sales by 1%, and others report that a 1-second slowdown reduces a customers satisfaction metric by 16%
- On the otherhand, the 99th percentile may be too expensive to improve and not yield enough benifits.


#### Service level objectives(SLO) and service level agreements(SLA)

Are contracts that define the expected performance and availability of a service:  
SLA - may state a service is up if the median response time is less than 200ms and a 99th percentile under 1 second   
If the SLA is not met, the service can be considered down, and a customer can demand a refund  

#### Percentiles in practice

High percentiles become important in backend services that are called multiple times as part of a serving a single end-user request. Even if the call is in parallel, the end-user will have to wait on the slowest backend service. Even if only a small percentage of backend calls are slow, the chances of getting a slow call increases if an end-user request requires mutiple back-end calls, and so a higher proportion of end-users requests end up being slow (effect is known as tail latency amplification)

#### Aproaches for coping with load

How do we maintain good performance even when our load parameters increase by some amount?

An architecture that is appropriate for one level of load is unlikely to cope with 10 times the load. You will need to rethink your architecture on every order of magnitude load increase. Popular terms are:

- Scaling up: (vertical scaling): adding more resources to an existing machine
- Scaling out: (horizontal scaling): add more machines into your pool of resources
- Shard nothing architecture: when a distributing load across multiple machines 

A good architecture involes a pragmatic mixture of approaches. For example, using serveral fairly powerful machines can be simple and cheaper than a large number of small virtual machines.  

Ways to scale:  
- elastic: system can automatically add computing resources when they detect a load increase
    - useful when load is highly unpredictable
- manual: system is analyzed and scaled manually
    - tend to be simpler and have fewer operational surprises
    
It used to be that distributing a stateful data system from a single node to a distributed setup can introduce a lot of complexity. It would only be done when scaling up cost exceed or high availability requirements force you to scale out into a distributed system,

AWS services makes scaling out very easy on users and is contributing to the boom in big data.

There is no one-size-fits-all on scalable architectures. This is because the volume or reads, the volume of writes, the volume of data to store, the complexity of the data, the response time requirements, the access patterns, or some mixture all of these plus many more issues are not directly compatiable. For example, a system that is designed to handle 100,000 request per second, each 1 kB in size looks very different from a system that is designed for 3 requests per minute, each 2 GB in size, even though the two systems have the same data throughput.  

##### Production System:  
**An architecture is built ontop of assumptions of which operations will be common and which will be rare - the load parameters. If those assumptions turn out to be wrong, the engineering effort for scaling is at best wasted, and at worst counterproductive.** 

##### Start-up System
**In an early-stage startup or an unproven product it's usually more important to be able to iterate quickly on product features than it is to scale to some hypothetical future load.**

#### Maintainability

The majoirty of the cost of sotware is not in its initial development, but in its ongoing maintenance - fixing bugs, keeping its systems operational, investigating failures, adapting it to new platforms, modifying it for new use cases, repaying technical debt, and adding new features.  

Yet, unfortunately, many people working on software systems dislike maintenance of so-called legacy systems-perhaps it involves fixing other people's mistakes, or working with platforms that are now outdated, or systems that were forced to do things they were never intended for. Every legacy system is unpleasant in its own way, and so it is difficult to give general recommendations for dealing with them.

However, we can and should design software in such a way that it will hopefully minimize pain during maintenance, and thus avoid creating legacy software outselves. To this end, we will pay particular attention to three design principles for software systems:  

- Operability: makes it easy for operation teams to keep the system running smoothly
- Simplicity: makes it easy for new engineeers to understand the system, by removing as much complexity as possible from the system
- Evolvability: makes it easy for engineers to make changes to the system in the future, adapting it for unanticipated uses cases as requirements change. Also know as extensibility, modifiability or plasticity

There is no easy soltion for achieving these goals.  

#### Operability: making life easy for operations

It has been suggested that good operations can often work around the limitations of bad software, but good software cannot run reliably with bad operations. Operation teams are vital to keeping a software system running smoothly. A good operation team typically is responsible for the following and more:  

- Monitoring the health of the system and quickly resoring services if it goes into a bad state
- Tracking down the cause of problems, such as system failures or degraded performance
- Keeping software and platforms up to date, including security patches
- Keeping tabs on how different systems affact each other, so that a problematic change can be avoided before it causes damage
- Anticipating future problems and solving them before they occur (capacity planning)
- Establishing good practices and tools for deployment, configuration management, and more
- Performing complex maintenance tasks, such as moving an application from one platform to another
- Maintaining the security of the system as configuration changes are made
- Defining processes that make operations predictable and help keep the production enviornment stable
- Preserving the organization's knowledge about the system, even as individual people come and go

Good operability means making routine tasks easy, allowing the operations teams to focus their effors on high-value activities. Data systems can do various things to make routine task easy, including:  

- Providing visibility into the runtime behavior and internals of the system, with good monitoring
- Providing good support for automation and integration with standard tools
- Avoiding dependency on individual machines (allowing machines to be taken down for maintenance while the system as a whole continues running uninterrupted)
- Providing good documentation and an easy-to-understand operational model (if I do X, Y will happen)
- Providing good default behavior, but also giving administrators the freedom to override defaults when needed
- Self-healing where appropriate, but also giving administrators manual control over the system state when needed
- Exhibiting predictable behavior, minimizing surprises

#### Simplicity: Mangaging Complexity

Small software projects can have simple and expressive code, but larger projects become very complex and difficult to understand. This complexity slows down everyone who needs to work on the system, further increaseing the cost of maintenance. Software projects mired in complexity is sometimes described as a *big ball of mud*.

Possible symptoms of complexity:  
- explosion of the state space
- tight coupling of modules
- tangled dependencies
- inconsistent naming and terminology
- hacks aimed at solving performance problems
- special-casing to wrok around issues elsewhere
- many more

Complexity makes:  
- maintenance hard with budgets
- schedules are often overrun
- greater risk of introducing bugs
- harder to understand and reason about
- has hidden assumptions
- unintended consequences
- unexpected interactions are more likely to be overlooked

Making a system simpler does not necessarily mean reducing its functionality, it can also mean removing accidental complexity.  

Tools for removing accidental complexity are good abstractions.  
- hide a great deal of implementation detail behind a clean, simple-to-understand front-end
- code reuse: can be used for a wide range of different applications for

Good abstractions for distributed systems are hard to find and package together to keep the complexity of a system managable.  

#### Evolvabilty: Making Change Easy

A system will likely always be in constant flux:  
- you learn new facts
- previewously unanticipated use cases emerge
- business priorities change
- users request new features
- new platforms replace old platforms
- legal or regulatory requirements change
- growth of the system forces architecture change
- etc

In terms of organizational processes, *Agile* working patterns provide a framework for adapting to change. The Agile community has developed technical tools and paterns that are helpful when developing software in a frequently changing enviornment, such as test-driven development (TDD) and refactoring. Most agile techniques focus on fairly small and local teams working on an application.  

The ease at which you can modify a data system, and adapt it to changing requirements, is closely linked to its simplicity and its abstractions. This term refering to the agility on modifying a data system is called evolvability.  

#### Summary

An application has to meet various requirements in order to be useful. There are functional requirements (what it should do, such as allowing data to be stored, retrieved, searched, and processed in various ways), and some nonfunctional requirements (general properties like secuirty, reliability, compliance, scalability, compatibility, and maintainability). In this chapter reliability, scalability, and maintainability were discussed in detail.  

Reliability means making systems work correctly, even when faults occur. Faults can be in hardware (typically random and uncorrelated), software (bugs are typically stematic and hard to deal with), and humans (who inevitably make mistakes). Fault-tolerance techniques can hide certain types of faults from the end users.  

Scalability means having strategies for keeping performance good, even when load increases. in order to discuss scalability, we first need ways of describing load and performance quantitatively. We briefly looked at Twitters's home timelines as an example of describing load, and response time percentiles as a way of measuring performance. In scalable systems, you can add processing capacity in order to remain reliable under high load.  

Maintainability has many facets, but in essence it is about making life better for engineering and operations teams who need to work with the system. Good abstractions can help reduce complexity and make the ststem easier to modify and adapt for new use cases. Good operability means having good visibility into the system's health, and having effective ways of managing it.  

There is unfortunately no easy fix for making applications reliable, scalable, or maintainable. However, there are certain patterns and techniques that keep reappearing in different kinds of applicatinos. In the next few chaperts we will take a look at some examples of data systems and analze how they work toward those goals.  


#### Whiteboard notes

![part1](https://raw.githubusercontent.com/JonathanWamsley/Designing-Data-Intensive-Applications/master/images/part%201.jpg)
![part2](https://raw.githubusercontent.com/JonathanWamsley/Designing-Data-Intensive-Applications/master/images/part%202.jpg)
![part3](https://raw.githubusercontent.com/JonathanWamsley/Designing-Data-Intensive-Applications/master/images/part%203.jpg)
![part4](https://raw.githubusercontent.com/JonathanWamsley/Designing-Data-Intensive-Applications/master/images/part%204.jpg)
![part5](https://raw.githubusercontent.com/JonathanWamsley/Designing-Data-Intensive-Applications/master/images/part%205.jpg)
![part6](https://raw.githubusercontent.com/JonathanWamsley/Designing-Data-Intensive-Applications/master/images/part%206.jpg)
![part7](https://raw.githubusercontent.com/JonathanWamsley/Designing-Data-Intensive-Applications/master/images/part%207.jpg)
![other](https://raw.githubusercontent.com/JonathanWamsley/Designing-Data-Intensive-Applications/master/images/other.jpg)