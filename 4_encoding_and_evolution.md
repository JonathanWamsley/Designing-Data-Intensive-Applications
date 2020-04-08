# Encoding and Evolution

Applications inevitably change over time. Features are added or modified as new products are launched, user requirements become better understood, or buisness circumsances change. In Chapter 1 we introduced the idea of evolvability: we should aim to build systemms that make it easy to adapt to change (see "Evolvability: Making Change Easy" on page 21).  

In most cases, a change to an application
s features also requires a change to data that it stores: perhaps a new field or record type needs to be captured, perhaps existing data needs to be presented in a new way.  

The data models we discussed in Chapter 2 have different ways of coping with such change. Relational databases generally assume that all data in the database conforms to one scheme: although that schema can be changed (through schema migrations; i.e., ALTER statements), there is exactly one schema in force at any one point in time. By contrast, schema-on-read ("schemaless") datavases don't enforces a schema, so the database can contain a mixture of older and newer data formats written at different times (see "schema flexibility in the document model" on page 39).  

When a data format or schema changes, a corresponding change to application code often needs to happen (for example, you add a new field to a record, and the application code starts reading and writing that field). However, in a large application, code changes often cannot happen instantaneously:  
- with server-side applications you want to perform a rolling upgrade (also known as a staged rollout), deploying the new version to a few nodes at a time, checking whether the new version is running smoothly, and gradually working your way through all the nodesl This allows new version to a few nodes at a time, checking whether the new version is running smoothly, and gradually working your way through all the nodes. This allows new versions to be deployed without service downtime, and thus encourages more frequent releases and better evolvabilit.  
- with client-side applications you're at the mercy of the user, who may not install the update for some time.  

This means that old and new versions of the code, and old and new data formats, may potentially all coexist in the system at the same time. In order for the system to continue running smoothly, we need to maintain compatibility in both directions:  

- Backward compatibility
    - Newer code can read data that was written by older code.  
    
- Forward compatibility
    - Older code can read data that was written by newer coder.
    
Backwards compatibility is normally not hard to achieve: as author of the newer code, you know the format of data written by older code, and so you can explicitly handle it (if necessary by simply keeping the old code to read the old data). Forward compatibility can be trickier, because it requires older code to ignore additions made by a new version of code.  

In this chapter we will look at several formats for encoding data, including JSON, XML, Protocol Buggers, Thrift, and Avro. In particular, we will look at how they handle schema changes and how they support systems where old and new sata and code needs to coexist. We will then discuss how those formats are used for data storage and for communication: in web services, Representational State Transer (REST), and remote procedure calls (RPC), as well as message-passing systems such as actors and message queues.  

### Formats for Encoding Data

Programs usually work with data in (at least) two different representations:  
1. In memory, data is kept in objects, structs, list, arrays, hash tables, trees, ans so on. These sata structures are optimized for efficient access and manipulation by the CPU (typically using pointers).
2. When you want to write data to a file or send it over the network, you have to encode it as some kind of self-contained sequence of bytes (for example, a JSON document), Since a pointer wouldn't make sense to any other process, this sequence-of-bytes representation looks quite different from the data structures that are normally used in memory.  

Thus, we need some kind of translation between the two representations. The translation from the in-memory representation to a byte sequence is called encoding (also known as serialization or marshalling), and the reverse is called decoding (parsing, deserialization, unmarshalling).  

As this is such a common problem, there are a myrid different libaries and encoding formats to choose from. Let's do a brief overview.  

### Language-Specific Formats

Many programming languages come with built-in support for encoding in=memory objects into byte sequences. For example, Java has java.io.Serializable, Ruby has Marshal, Python has pickle, and so on. Many third-party libraries also exist, such as Kryo for Java.  

These encoding libraries are very convenient, because they allow in-memory objects to be saved and restored with minimal additional code. However they also have a number of deep problems:  

- The encoding is often tied to a particular programming language, and reading the data in another language is very difficult. If you store or transmit data in such an encoding, you are commiting yourself to your current programming language for potentially a very long time, and precluding integrating your systems with those of other organizations (which may use different languages).  
- In order to restore data in the same object types, the decoding process needs to be able to instantiate arbitrary classes. This is frequently a source of security problems: if an atterker can get your application to decode an arbitrary byte sequence, the can instantiate arbitrary classe, which in turn often allows them to do terrible things such as remotely executing arbitrary code.  
- Versoning data is often an afterthought in these libraries: as they are intended for quick and easy encoding of data, they often neglect the inconvenient problems of forward compatibility.  
- Efficiency (CPU time taken to encode or decode, and the size of the encoded structure) is also often an afterthought. For example, Java's built-in serilization is notorious for its bad performance and bloated encoding.  

For these reasons it's generally a bad idea to use your language's built-in encoding for anything other than very transient purposes.  

### JSON, XML, and Binary Variants

Moving to standardized encodings that can be written and read by many programming languages, JSON and XML are the obvious contenders. They are widely known, widely supported, and almost as widely disliked. XML is often criticized for being too verbose and unnecessarily complicated. JSON's popularity is mainly due to its built-in support in web browsers (by virtye of being a subset of JavaScript) and simplicity relative to XML. CSV is another popular language-independent format, albeit less powerful.  

JSON, XML, and CSV are textual formats, and thus somewhat human-readable (although the syntax is a popular topic of debate). Besides the superficial syntactic issues, they also have some subtle problems:  
- There is a lot of ambiguity around the encoding of numbers. In XML and CSV, you cannot distinguish between a number and a string that happens to consist of digits (except by referring to an external schema) JSON distinguishes strings and numbers, but it doesn't distinguish integers and floating-point numbers, and it doesn't specify a precision.  
- This is a problem when dealing with large numbers; for example, integers greater than 2 cannot be exactly represented in an IEEE 754 double-precision floating-point number, so such numbers become inaccurate when parsed in a language that uses floating-point numbers (such as JavaScript). An example of numbers large than 2^53 occur on Twitter, which uses a 64-bit number to identify each tweet. The JSON returned by Twitter's API include tweet IDs twice, once as a JSON number and once as a decimal string, to work around the fact that the numbers are not correctly parsed by JavaScript applications.  
- JSON and XML have good support for Unicode character string (i.e., human readable text), but they don't support binary strings (sequences of bytes without acharacter encoding). Binary strings are a useful feature, so people get around thiss limitation by encoding the binary data as text using Based64. The schema is then used to indicate that the value should be interpreted as Base64-encoded. This works, but it's somewhat hackey and increases the sata size by 33%.  
- There is optional schema support for both XML and JSON. These schema languages are quite powerful, and thus quite complicated to learn and implement. Use of XML schemas is fairly widespread, but many JSON-based tools don't bother using schemas. Since the correct interpretation of data (such as numbers and binary strings) depends on information in the schema, applications that don't use XML/JSOn schemas need to potentially hardcode the appropriate encoding/decoding logic instead.  
- CSV does not have any schema, so it is up to the application to define the meaning of each row and column, If an application change adds a new row or column, you have to handle that change manually. CSV is also quite vague format (what happens if a value contains a comma or newline character?). Although its escaping rules have been formally specified, not all parsers implement them correctly.  

Despite these flaws, JSON, XML, and CSV are good enough for many purposes, It's likely that they will remain popular, especially as data interchange formats (i.e., for sending data from one organization to another). In these situations, as long as people agree on what the format is, it often doesn't matter how pretty or efficient the format is. The difficulty of getting different organizations to agree on anything outweights most other concerns.  

### binary encoding

For data that is used only internally within your organization, there is less pressure to use the lowest-common-denominator encoding format. For example, you could choose a gormat that is more compact or faster to parse. For a small dataset, the gains are negligible, but once tou get into the terabytes, the choice of data format can have big impact.  

JSON is less verbose than XML but both still use a lot of space compared to binary formats. THis observation led to the develpment of a profusion of binary encodings for JSON (MessagePack, BSON, BJSON, UBJSON, BIJSON, and Smile, to name a few) and for XML (WBXML anf Fast Infoset, for example). These formats have been adopted in various niches, but none of them are widely adopted as the textual versions of JSON and XML.  

Some of these formats extend the set of datatypes (e.g., distinguishing integers and floating point numbers, or adding support for binary strings), but otherwise they keep the JSON/XML data model unchanged. In particular, since they don't prescribe a schema, they need to include all the object field names within the encoded data. That is in a binary encoding of the JSON document in Example 4-1, they will need to include the strings userName, favoriteNumber, and interests somewhere.  

<p>
    Example 4-1. Example record which we will encode in several binary formats in this chapter  
    
    {
        "userName": "Martin",
        "favoriteNummber": 1337,
        "interests": ["daydreaming", "hacking"]
    }
</p>

Let's look at an example of MessagePack, a binary encoding for JSON. Figure 4-1 shows the byte sequence that you get if you encode the JSON document in Example 4-1 with MessagePack. The first few bytes are as follows:  

1. The first byte, 0x88, indicates that what follows is an object (top four bits - 0x80) with three fields (bottom four bits = 0x03). (in case you're wondering what happens if an object has more than 15 fields, so that the nuber of fields doesn't first in four bits, it then gets a different type indicator, and the number of field is encoded in two or four bytes.)  
2. The second byte, 0xa8, indicates that what follows is a string (top four bits - 0xa0) that is eight bytes long (bottom four bits = 0x08).  
3. The next eight bytes are the field name userName in ASXII. Since the length was indicated previously, there's no need for any marker to tell us where the string ends (or any escaping).  
4. The next seven bytes encode the six0letter string value Martin with a prefix 0xa6, and so on.  

The binary encoding is 66 bytes long, which is only a little less than the 81 bytes taken by the textyal JSON encoding (with whitespace removed). All the binary encoding of JSON are similar in this regard. It's not clear whether such a small space reduction (and perhaps a speedup in parsing) is worth the loss of human-readability.  

In the following section we will see how we can do much better, and encode the same record in just 32 bytes.  

### Thrift and Protocol Buffers

Apache Thrift and Protocol Buffers (protobuf) are binary encoding libarries that are based on the same principle. Protocol Buffers was originally developed at Google, Thrift was originally developed at Facebook, and both were made open source in 2007-2008.  

Both Thrift and Protocol Buffers requre a schema for aby data that is encoded. To encode the data in Example 4-1 in Thrift, you would describe the schema in the Thrift interface definition language (IDL) like this:  

<p>
    struct Person {
        1: required string userName,
        2: optional i64 favoriteNumber,
        3: optional list string> interests
    
    
    The equivalent schema definition for Protocol Buggers looks very similar:
    
    message Person {
        required string user_name = 1;
        optional int 64 favorite_number = 2;
        repeated string interests = 3;
    }
</p>

Thirst and Protocol Buffers each come with code generation tool that takes a schema definition like the ones shown here, and produces classes that implement the schema in various programming languages. Your application code can call this generated code to encode or decode records of the schema.  

What does data encoded with this schema look like? Confusingly, Thrift has two0different binary encoding formats, called BinaryProtocol and CompactProtocol, respectively. Let's look at binaryProtocol first. Encoding Example 4-1 in that format takes 59 bytes, as shown in Figure 4-2.  

Similarly to Figure 4-1, each field has a type annotation (to indicate whether it is a string, integer, list etc.) and, where required, a length indication (length of a string, number of items in a list). The strings that appear in the data ("Martin", "daydreaming", "hacking") are also encoded as ASCII (or rather UTF-8), similar to before.  

The big difference compared to Figure 4-1 is that there are no field names (userName, favoriteNumber, interests). Instead, the encoded data contains field tags, which are numbers(1, 2, and 3). Those are the numbers that appear in the schema defination. Field tags are like aliases for fields - they are compact way of saying what field we're talking about, without having to spell out the field name.  

The thrift CompactionProtocol encoding is semantically equivalent to BinaryProtocol, but as you can see in Figure 4-3, it packs the same information into only 34 bytes. It does this by packing the field type and tag number into a single byte, and by using variable-length integers. Rather than using a full eight bytes for number 1337, it is encoded in two bytes, with the top bit of each byte used to indicate whether there are still more bytes to come. This means numbers between -64 and 63 are encoded in one byte, numbers between -8192 and 8191 are encoded in two bytes, etc. Bigger numbers use more bytes.  

Finally, Protocol Buffers (which has only one binary encoding format) encodes the same sata as shown in Figure 4-4. It does the bit packing slightly differently, but is otherwise very similar to Thrift's CompactProtocol. Protocol Buffers fits the same record in 33 bytes.  

One detail to note: in the schemas shown earlier, each field was marked either required or optional, but this makes no difference to how the field is encoded (nothing in the binary data indicates whether a field was required). The difference is simply that required enables a runtime check that fails if the field is not set, which can be useful for catching bugs.  

### Field tags and scheme evolution

We said previously that schemas inevitably need to change over time. We call this schema evolution. How do Thrift and Protocol Buffers handle schema changes while keeping backward and forward compatibility?  

As you can see from the examples, an encoded record is just the concatentation of its encoded fields. Each field is identified by its tag number (the numbers 1, 2, 3 in the sample schemas) and annotated with datatype (e.g., string or integers). If a field value is not set, it is simply omitted from the encoded record. From this you can see that field tags are critical to the meaning of the encoded data. You can change the name of a field in the schema, since the encoded data never refers to field names, but you cannot change a field's tag, since that would make all existing encoded data invalid.  

You can add new fields to the schema, provided that you give each field a new tag number. If old code (which doesn't know about the new tag numbers you added) tries to read data written by new code, including a new field with a tag number it doesn't recognize, it can simply ignore the field. The datatype annotation allows the parser to determine how many bytes it needs to skip. This maintains forward compatibility: old code can read records that were written by new code.  

What about backward compatibility? As long as each field has a unique tag number, new code can always read old data, because the tag numbers still have the same meaning. The only detail is that if you add a new field, you cannot make it required. If you were to add a field and make it required, that check would fail if new code read data written by old code, because the old code will not have written the new field that you added. Therefore, to maintain backward compatiability, every field you add afer the initial deployment of the schema must be optional or have a default value.  

Removing a field is just like adding a field, with backward and forward compatibility concerns reversed. That means you can only remove a field that is optional (a required field can never be removed), and you can never use the same tag number again (because you may still have data written somewhere that includes the old tag number, and that field must be ignored by new code).  

### Datatypes and schema evolution

What about changing the datatype of a field? that may be possible - check the documentation for details - but there is a risk that values will lose precision or get tuncated. For example, say you change a 32-bit integer into a 64-bit integer. New code can easily read data written by old code, beause the parser can fill in any missing bits with zeros. However, if old code reads data written by new code, the old code is still using a 32-bit variable to hold the value. If the decoded 64-bit value won't fit in 32 bits, it will be truncated.  

A curious detail of Protocol Buffers is that it does not have a list or array datatype, but instead has a repeated marker for fields (which is a third option alonggside required and optional). As you can see in Figure 4-4, the encoding of a repeated field is just what it says on the tin: the same field tag simply appears multiple times in the record. This has the nice effect that it's okay to change an optional (singlevalued) field into a repeated (multivalued) field. New code reading old data sees list with zero or one elements(depending on whether the field was present); old code reading new data sees only the last element of the list.  

Thrift has a dedicated list datatype, which is parameterized with datatype of the list elements. This does not allow the same evolution from single-valued to multivalued Protocol Buffers does, but it has the advantage of supporting nested lists.  

### Avro


