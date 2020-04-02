# Designing Data-Intensive Applications

By: Martin Kleppmann

# Preface

Data-intensive applications are pushing the boundaries of what is possible because:  
- Big companies are handling huge volumes of data and trafic which force them to create new tools to handle problems at scale
- Businesses need to be agile, test hypotheses cheaply, and respond quickly to new market insights by keeping development cycle short and data models flexible
- free and open source software has become very sucessful and preferred over commerical products
- CPU clock speeds are barely increasing, but multi-core processors are standard, and networks are getting faster. Parallelism is getting better.  
- small companies can build distributed systems thanks to infrastructure as a service(IaaS) because of AWS
- many services are no expected to be highly available, extended downtime due to outages or maintenance is becoming increasingly unacceptable

An application is data-intensive if data is its primary challenge- the quantity of data, the complexity of data, or the speed at which is changing (as opposed to compute-intensive where CPU is bottleneck)  

The goal of this book is to help you navigate the diverse and fast-changing landscape of techonolgies for processing and storing data. 

## Outline of this book

####  Part 1. dicuss fundamental ideas behind the design of intensive applications. 
1. chapert 1 define what we are actually trying to achieve reliablility, scalability, and maintainability; how we need to think about them, achieve them, etc.
2. chapter 2 compares serveral different data models and query languages and sees how they are appropriate to different situations
3. chapter 3 talks about storage engines: how databases arrange data on a disk so that we can find it again efficiently
4. chapter 4 turns to formats for data encoding(serialization) and evolution of schemas over time.

#### Part 2. moves from data stored on one machine to data that is distributed across multiple machines. This is often nesessary for scalability, but brings with it a variety of unique challenges. 
5. chapter 5: discusses replication
6. chapter 6: discusses partition
7. chapter 7: discuess transactions
8. chapter 8: more problems on distributed systems 
9. chapter 9: what is means to achieve consistency and consensus in a distributed system

#### Part 3. discusses system that derive some datasets from other datasets. Derived data often occurs in heterogeneous systemsL when there is no one data base that can do everything well, applications need to integrate serveral different databases, caches, indexes and so on.
10. chapter 10: discusses batch processing approach to derived data
11. chapter 11: discusses stream processing
12. chapter 12: large summary and combination of material and the future.