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

