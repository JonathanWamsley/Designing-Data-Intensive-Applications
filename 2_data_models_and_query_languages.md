# Data Models and Query Languages

Data models are perhaps the most important part of developing software, because they have such a profound effect on:
- how the software is written
- how we *think about the probelm* we are solving

Most applications are built by layering on data model on top of another. For each layer, the key questions is *represented* in terms of the next-lower layer, for example:  
1. As an applications developer, you look at the real world (in which there are people, organizations, goods, actions, money flows, sensors, etc.) and model it in terms of objects or data structures, and APIs that manipulate those data structures. Those structures are often specfic to your application.  
2. When you want to store those data structures, you express them in terms of a general-purpose data model, such as JSON or XML documents, tabless in a relational database, or a graph model.  
3. The engineers who built your database software decide on a way of representing that JSON/XML/relationship/graph data in terms of bytes in memory, on disk, or on a network. The representation may allow the data to be queried, searched, manupulated, and processed in various ways.
4. On yet lower levels, hardware engineers have figured out how to represent bytes in terms of electrical currents, pulses of light, magnetic fields, and more.  

In a complex application there may be more intermediary levels, such as APIs built upon APIs, but the basic idea is stil the same: each layer hides the complexity of the layers below it by providing a clean data model. These abstractions allow different groups of people - for example, the engineers at the datbase vendor and the applications developers using their database - to work together effectively.  

There are many different kinds of data models, and every day model embodies assumptions about how it is going to be used. They can be: 
- easy to use or not supported
- some operations are fast and some perform badly
- some data transformations feel natural and some are awkward  

It can take a lot of effort to master just one data model (think how many bookers there are on relational data modeling). Building software is hardenough, even when working wit just one data model and without worrying about its inner workings. Butsince the data model has such a profound effect on what the software aboive it can and can't do, it is important to choose one that is appropriate to the applications.  

This chapter looks at:
- range of general-purpose data models for data storage and querying
- compare the relational, document, and graph-based data models
- look at various query languages and compare their use cases

### Relational Model Versus Document Model

The best-known data model today is SQL based on a relational model. The data is organized into relations called tables that are unordered collections of tuples(rows in SQL). The relational database was originally useful in business data processing to perform transactional processing and batch processing. 
- transactional processing: entering sales or banking transactions, airline reservations, stock-keeping in warehouses
- batch processing: customer invoicing, payroll, reporting

As computers became more powerful and networked, relational databased started being used for diverse purposes and performed well. Relational databases were used for tasking includeing:
- online publishing
- social networking
- ecomerce, games
- software-as-a-service productivity applications 
- and many more

### The Birth of NoSQL

In 2010, NoSQL started getting popular through web startup communities and beyond. Reasons for adopting the NoSQL database include:
- a need for greater scalability than relational databases can easily achieve, including very large datasets or very high write throughput
- a widespread preference for free and open source software over commercial products
- specialized query operations that are not well supported by the relational model
- frustration with the restrictiveness of relational schemas, and a desire for a more dynamc and expressive data model  

Different applications have different requirements, and the best choice of technology for one use case may web be different from the best choice for anotehr use case. It therefore seems likely that in the foreseeable future, relational databases will continue to be used alongside a broad variety of nonrelational datastores - an idea that is called *polyglot persistence*  

### The Object-Relational Mismatch  

Most applications development today is done in object-oriented programming languages, which leads to a common criticism of the SQL data model:
> if data is stored in relational tables, an awkward translation layer is required between the objects in the application code and the database model of tables, rows, and columns. This disconnect is known as *impedance mismatch*  

The book gives an example of a resume profile. A relational schema would need a unique identifier (user_id), then have several tables that contain many-to-many relationships. There would be tables for user, region, industries, table, position, education, contact info, etc.  

For a datastructure like a resume, it is mostly a self-contained document, and a JSON representation can model the data in a simple way. A document-oriented database like MongoDB, Amazon DocumentDB, CouchDB are examples. Some developers feel that JSON model reduces the impedance mismatch between the application code and the storage layer. However in Chapter 4, there are problems with JSON as data encoding format. The lack of a schema is often cited as an advantage.  

The JSON model representation ahs better *locallity* than the multi-table schema. If you want to fetch a profile in the relational example, you need to either perform multiple queries (query each table by user_id) or perform a messy multiway join between the user table and its subordinate tables. In the JSON representaion, all teh relevant information is in one place, and one query is sufficient.  

The one-to-many relationships from the user profile to the user's positions, educational history, and contact information imply a tree structure in the data, and the JSON representaion makes this tree structure explicity.  

### Many-to-One and Many-to-Many Relationships

In the preceding section, region_id and industry_id are given as Ids, not as plain text strings "Greater Seattle Area" and "Philanthropy". Why?  

If the user interface has free-text fields for entering the region and the industry, it makes sense to store them as plain-text strings. But there are advantages to having standaraized lists of geographic regions and industries, and letting users choose from a drop-down list or autocompleter:  
- consistent style and spelling across profiles
- avoid ambiguity (if there are several cities with the same name)
- ease of updating- the name is stored in only one place, so it is easy to update across the board if it ever needs to be changed
- localization support - when the site is translated into other languages, the standardized lists can be localized, so the region and industry can be displayed in the viewer's language
- better search - a search for philanthropists in the state of washinton can match this profilem because the list of regions can envode the fact that Seattle is in Washinton (which is not apparent from the string 'Greater Seattle Area')  

Whether you store an ID or a text string is a question of duplication. When you use an ID, the information that is meaningful to humans (such as the word Philanthropy) is stored in only one place, and everything that refers to it uses an ID (which only has meaning within the database). When you store the text directly, you are duplicating the human-meaninful information in every record that uses it.  

The advantage of using an ID is that because it has no meaning to humans, it never needs to be changeL the ID can remain the same, even if the information it identifies to changes. Anything that is meaningful to humans may need to be change sometime in the future - and if that information is duplicated, all the redundant copies need to be updated. That incurs write overheads, and risks inconsistencies (where some copies of the information are updated but others are not). Removing such duplication is the key idea behind *normalization* in databases. 

Unfortunately, normalizing this data requires many-to-one relationships (many people live in one particular region, many people work in one particular industry), which don't fit nicely into the document model. In a relational databases, it is normal to refer to rows in other tables by ID, because joins are easy. In document databases, joins are not needed for one-to-many tree structures, and support for joins is often weak. 
- document can do one-to-many but not many-to-one relationships well

If the database does not support joins, you have to emulate a join in application code by making multiple queries to the database.  Moreover, even if the initial version of an application fits well in a join-free document model, data has a tendency of becoming more interconnected as features are added to applications. For example if a resume added changes:
- adding entities like organizations or schools
    - each would have their own webpage with table like information
- adding features like a recommendation section
    - if one user can write a recommendation for another user. the recommendation should have a reference to the author's profile, name, photo, etc
    
The key point is many-to-many relationships can be done in document db but it creates complexities with multiple queries to join information together.  

### Are Document Databases Repeating History?

While many-to-many relationships and joins are routinely used in relational databases, document databases and NoSQL reopened the debate on how best to represent such relationships in a database.  

Database types have trade offs in their abilities to model relationships betwen:
- one-to-many
- many-to-one
- many-to-many

### The relational model

The relational model layed out all the data in a relational table as a collection of tuples. No nested structuresm not complicated access paths to follow. You can read any or all of the rows in a table, selecting those that match an arbitrary condition. You can read a particular row by designating some columns as a key and matching on those. You can insert a new row into any table without worrying about foreign key relationships to and from other tables.  

In a relational database, the query optimizer automatically decides which part of the query to execute in which order, and which indexes to use. Those choices are effectively the access path done automatically.  

If you want to query your data in new ways, you can just declare a new index, and queries will automatically use whichever indexes are most appropriate. Thus relatinoal models made it easier to add new features to applications.  

Query optimizers for relational databases are complicated beasts, and they have consumed many years of research and development effort. But a key insigh of the relational model was this: you only need to build a query optimizer once, and then all applications that use the database can benefit from it.  


### Comparison to document databases

Document databases reverted back to the hierachial model in storing nested records. When it comes to representing many-to-one and many-to-many relationships, relational and document databases are not fundamentally different: in both cases, the related item is referenced by a unique identifier, which is called a foriegn key in a relational mode and a document referenece in a document model. That identifier is resolved at read time by using a join or follow-up query.  

### Relational Versus Document Databases Today

The main arguments in favor of the document data model are schema flexibility, better performance due to locality, and that for some applications it is closer to the data structures used by the application. The relational model counters by providing better support for joins and many-to-one and many-to-many relationships.  

### Which data model leads to simpler application code?  


If the data is your application has a document-like structure (i.e., a tree of one-to-many relationships, where typically the entire tree is loaded at once), then its probabily a good idea to use a document model. The relational technique of shredding- splitting a document-like structure into multiple tables (like position, education, and contact_info) can lead to cumbersome schemas and unnecessarily complicated application code.  

The document model has its limitations: you cannot refer directly to a nested item with a document, but instead you need to say something like "the second item of osition for user 251". However, as long as documents are not too deeply nested, that is not usually a problem. If your application does use many-to-many relationships, the document model becomes less appealing. 

### Schema flexibility

Most document databases, and the JSON support in relational databases, do not enforce any schema on the data in documents. XML support in relational databases usually comes with optional schema validation. No schema means that arbitrary keys and values can be added to a document, and when reading, cllients have no guarantees as to what fields the documents may contains.  

Document databases are sometimes called *schemaless*, but that is misleading, as the code that reads the data usually assumes some kind of structure, there is an implicit schema, but it is not enforced by the database. A more accurate term is *schema-on-read* (the structure of the data is implicit, and only interpreted when the data is read), in contrast with *schema-on-write* (the traditional approach of relational databases where the schema is explicit and the database ensures all written data conforms to it).  


Schema-on-read is similar to dynamic type checking in programming languages, whereas schema-on-write is similar to static type checking. Just as the advocates of static and dynamic type checking have big debates about their relative merits, enforeent of schemas in database is contentiuos topic, and in general there is no right or wrong answer.  

The difference between the approaches is particularly notiable in situations where an application wants to change the format of its data. For example, say you are currently storing each user's full name in one field, and you instead want to store that first name and last name separarately. In a document database, you would just start writing new documents with the new fields and have code in the application that handles the case when old documents are read. For example:  

> if (user && user.name && !user.first_name) {  
>    // Document written before Dec 8, 2013 don't have first_name  
>    user.first_name = user.name.split(" ")[0]  
> }  

On the other hand, in a "statically typed" database schema, you would typically perform a migration along the lines of:

> ALTER TABLE users ADD COLUMN first_name text;  
> UPDATE users SET first_name = split_part(name, ' ', 1); --PostgresSQL  

Schema changes have a bad reputation of being slow and requiring downtime. This reputation is not entirely deserved: most relational databases systems execute the ALTER TABLE statement in a few miliseconds.  

Running the UPDATE statement on a large table is likely to be a slow on any database, since every row needs to be rewritten. If that is not acceptable, The application can leave first_name set to its default of NULL and fill it in at read time, like it would with a document database.  

The schema-on-read approach is advantageous if the item in the collection don't all have the same structure for some reason:  
- there are many different types of objects, and it is not practival to put each type of object in its own table  
- the structure of the data is determined by external systems over which you have no control and which many change at any time  

In situations like that, these schemas may hurt more than it helps. But in cases where all records are expected to have the same structure, schemas are a useful mechanism for documenting and enforcing that structure.  

### Data locallity for queries  

A document is usually storedas a single continous string, encoded as JSON, XML, or a binary variat. If your application often needs to access the entire document (for example, to render it on a webpage), there is a performance advantage in this storage locality. If data is split across multiple tables, multiple index lookups are required to retrieve it all, which may require more disk seeks and take more time.  

The locality advantage only applies if you need large parts of the document at the same time. The database typically needs to load the entire document, even if you access only a small portion of it, which can be wasteful on large documents. On updates to a document, the entire document usually needs to be rewritten - only modifications that don;t change the encoded size of a document can easily be performed in palce. For these reasons, it is genereally recommended that you keep documents fairly small and avoid writes that increase the size of a document. These performances limitations significantly reduce the set of situations in which document databases are useful.  

It is worth pointing out that the idea of grouping related data together for locality is not limited to the document mode. For example, Google's Spanner database offers the same locality properties in a relational data model, by allowing the schema tot declare that a table's row should be interleaved with a parent table. Oracle allows the same, using a feature called multi-table index cluster tables. The column-family concept in the Bigtable data model (used in Cassandra and HBase)  has a similar purpose of managing locality.  

### Convergence of document and relational databases  

Most relational database systems have supported XML since the mid 2000's. This includes functions to make local modifications to XML documents and the ability to index the query inside XML documents, which allows applications to use data models very similar to what they would do when using a document database. 

Many databases also support JSON documents. Given the popularity of JSON for web APIs, it is likely that other relational databases will follow in their fotsteps and add JSON support.  

On the document database side, RethingDB supports relational-like joins in its query language, and some MongoDB drivers automatically resolve database references (effectively performing a client-side join, although this is likely to be slower than a join performed in the database since it requires additional network round-trips and is less optimized).  

Relational and document databases are becoming more similar over time, and that is a good thing: the data models complement each other. If a database is able to handle document-like data and perform relational queries on it, applications can use the combination of features that best fits their needs.  

A hybrid of the relational and document model is a good route for databases to take in the future.  

### Query language for data

When a relational model is introduced, it included a new way of querying data: SQL is a declarative query language, whereas most programming languages are imperative. For example, if you have a list of animals species, you might write soemthing like this to return only the sharks in the list:  

Function getSharks() {  
var sharks = [];  
for (var i = 0;i < animals.length; i++) {  
    if (animals[i].family == "Sharks") {  
        sharks.push(animals[i]);  
    }  
}  
return sharks;  
}  


In relational algebra, you would instead write:  
sharks = simga_family = "Sharks"^(animals)  

where sigma is the selection operator, returning those animals that match the condition family = "Sharks".  

When SQL was defined, it followed the structure of the relational algebra fairly closely:  

SELECT * FROM animals WHERE family = 'Sharks';  

An imperative language tells the computer to perform certain operations in a certain order. You can imagine stepping through the code line by line, evaluating conditions, updating variables, and deciding whether to go around the loop one more time.  

In a declarative query language, like SQL or relational algebra, you just specify the pattern of the data you want = what condition the results must meet, and how you want the data to be tramsformed (sorted, grouped, aggregated) - but not how to achieve that goal. It is up to the database system's query optimizer to decide which indexes and which join methods to use, and in which order to execute various parts of the query.  

A declarative query language is attractive because it is typically more concise and easier to work with than an imperative API. But more importantly, it also hides implementation details of the database engine, which makes it possible for the database system t introduce performance improvements without requireing any changes to queries.  

For example, in the imperative code shown at the beginning of this section, the list of animals appears in a particular order. If the database wants to reclaim unused disk space behind the scenes, it might need to move records around, changing the order in which the animals appear. Can the database do that safely, without breaking queries?  

The SQL example does not guarantee any particular ordering, and so it does not mind if the order changes. But if the query is written as imperative code, the database can never be sure whether the code is relaying on the ordering or not. The fact that SQL is more limited in functionality gives the database much more room for automatic optimization.  

Finally, declaritive languages often lend themselves to parallel execution. Todaym CPUs are getting faster by adding more cores, not by running at significantly higher clock speeds than before. Imperative code is very hard to parallelize across multiple cores and multiple machines, because it specifies instructions that must be performed in a particular order. Declarative language have a better change of getting faster in parallel execution because they specify only the pattern of the results, not the algorithm that is used to determine the results. The database is free to use a parallel implementation of the query language, if appropriate.  

### MapReduce Querying

MapReduce is a programming model for processing large amounts of data in bulk across many machines, popularized by Google. A limited fro mof MapReduce is supported by some NoSQL datastores, including MongoDB and CouchDB, as a mechanism for performing read-only queries across many documents.  

MapReduce in general is described in more detail in chapter 10. For now, we will just briefly discuss MongoDB's use of the model.  

MapReduce is neither a declarative query language nor a fully imperative query API, but somewhere in between: the logic of the query is expressed with snippets of code, which are called repeatedly by the processing framework. It is based on the map (also known as collect) and reduce (also known as fold or inject) functions that exit in many functional programming languages.  

To give an example, imagine you are a marine biologist, and you add an abservation record to your database everytime you see animals in the ocean. Now you want to generate a report saying how many sharks you have sighted per month.  

In PostgreSQL you might express that query like this:  

SELECT data_trunc('month', 'observation_timesamp') AS observation_month, sum(num_aninmals) AS total_animals  
FROM observations  
WHERE family == 'Sharks'  
GROUP BY observation_month;  

(1) The data_trunc('month', 'observation_timesamp')  function determines the calender month containing timestamp, and returns another timestamp representing the begining of that month. In other words, it rounds a timestamp down the the nearest month.  

The query first filters the observations to only show species in the sharks family, then groups the obervations by the calender month in which they occured, and finally adds up the number of aninmals seen in all observations in that month.  

The same can be expressed with MongoDB's MapReduce feature as follows:  

<p>
    db.observation.mapReduce(
        function map() { (2)
            var year = this.observationTimestamp.getFullYear():
            var month = this.observationTimestamp.getMonth() + 1;
            emit(year + "-" + month, this.numAnimals); (3)
        },
        function reduce(key, value) { (4)
            return Array.sum(values); (5)
        },
        {
            query; {family: 'sharks'}, (1)
            out: "monthlySharkReport" (6)
        }
    };
    
</p>

1. The filter to consider only shark species can be specified declaratively (this is a MongoDB-specific extension to MapReduce).  
2. The JavaScript function map is called once for every document that matches query, with this set to the document object. 
3. The map function emits a key (a string consisting of year and month, such as '2013-12' or '2014-1') and a value (the number of animals in that observation).  
4. The key-value pair emmited by map are grouped by key. For all key-value pairs with the same key (i.e. the same month and year), the reduce function is called once.  
5. The reduce function adds up the number of animals from all observations in a particular month.  
6. The final output is written to the collection monthlySharkReports.  

The map and reduce functions are somewhat restricted in what they are allowed to do. They must be pure functions, which means they only use the data that is passed to them as input, they cannot perform additional database queries, and they must not have any side effects. These restrictions allow the database to run the functions any where, in any order, and rereun them on failure. However, they are nevertheless powerful: they can parse strings, call library functions, perform calculations, and more.  

MapReduce is a failry low-level porgramming model for distributed execution on a cluster of machines. Higher-level query languages like SQL can be implemented as a pipeline of MapReduce operations (see chapter 10), but tehere are also many distributed implementations of SQL that don't use MapReduce. Note there is nothing in SQL that constrains it to running on a single machine, and MapReduce does not have a monopoly on distributed query execution.  

Being able to use JavaScript code in the middle of query is great feature for advanced queries, but it is not limited to MapReduce - some SQL databases can be extended with HavaScript functions too.  

A usability problem with MapReduce is that you have to write two carefully coordinated JavaScript functions, which is often harder than writing a single query. Moreover, a declarative query language offers more opportunities for query optimizer to improve the performance of a query. For these reasonsm MongoDB 2.2 added support for a declarative query language called the aggregation pipeline.  

The aggregation pipeline is similar in expressiveness to a subset of SQL, but it uses a JSON-based syntax rather than SQL's English-sentence-style syntax; the difference is perhaps a matter of taste. The moral of the story is that a NoSQL system may find itself accidentally reinventing SQL, albeit in disguise.  

#### Graph-like Data models

skipped for now.


### Summary 

Data models are a huge subject, and in this chapter we have taken a quick look at a broad variety of different models. WE did not have space to go into all the details of each model, but hopefully the overview has been enough to whet your appetite to find out more about the model that best fits your application's requirements.  

Historically, data started out being represented as one big tree (the hierarchial model), but that was not good for representing many to many relationships, so the relational model was invented to solve the problems. More recently, developers found that some applications don't fit well in the relational model either. New nonrelational NoSQL datastores have diverged in two main directions:  
1. Document databases target use cases where data comes in self-contained documents and relationships between one document and another are rare.  
2. Graph databases go in the opposite direction, targeting use cases where anything is potentially related to everything.  

All three models are widely used today, and each is good in its respective domain. One model can be emulated in terms of another model - for example, a graph data can be represented in a relational database -but the result is often awkward. That is why we have different systems for different purposes, not a single one-size-fits-all solution.  

One thing that document and graph databases have in common is that they typically don't enforce aschema for the data they store, which can make it easier to adapt applications to changing requirements. Howeverm your application most likely still assume that data has a certain structure; it's just a question of whether the schema is explicit(enforcced on writes) or implicit (handled on read).  





