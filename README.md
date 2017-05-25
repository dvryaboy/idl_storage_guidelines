# Schema best practices for long-term data storage

This document attempts to capture useful patterns and warn about subtle gotchas when it comes to designing and evolving schemas for long-term serialized data. It is not intended as a guide for how to best represent a particular dataset or process.

It assumes the reader intends to store data described by an IDL such as Apache Thrift, Apache Avro, or Protocol Buffers. If you are not familiar with any of those projects, you may wish to first familarize yourself with them, although we tried to keep this document fairly accessible to a technical reader who does not have extensive experience with these frameworks.

## Part 1: Preliminaries 

### Why Thrift, Protocol Buffers, or Avro?

When storing data in a KV store, Hadoop, or another medium that does not require strong up-front schemas (unlike, say, an SQL database), one has a variety of format options in which to store that data.

Dumping plain text into files is one option. Various comma- or tab-delimited files are popular. JSON may seem like a reasonable choice.  

It is the considered opinion of the authors of this document that if you can get away with it -- there are legitimate cases when you cannot -- it is far better to use a formal IDL to describe your data than to use an adhoc markup, or, worse yet, free-form text.

Verifiable schema declaration ensures changes to schema are recorded, versioned, auditable, and reviewable. Schemas can also be enforced on write, preventing a lot of headaches down the line. Analysts can search and examine schemas without the need to attempt to reconstruct what is possible from samples. Schema separable from data is a good thing. Field name typos are impossible.

Compact binary representations exist for these formats; this can mean significant savings for large datasets without the read and write overhead of generic compression algorithms. Straightforward conversion to formats like Parquet if you want even better queryability / compression is possible.

If, on the other hand, you wish to stay away from binary formats, and have been staying away from Thrift/PB/Avro for that reason, you can always choose use JSON as the serialization format, but still use code-genned libraries that enforce schemas to read and write your data.

### Why “for long-term data storage”?

Some patterns called out below are motivated by the need to provide a straightforward path to evolving the schema. Schema evolution is significantly simpler when Thrift and friends are used for RPC between services, and one can have reasonable expectations about the age of deployed message-producing libraries. Unless large data re-writes are frequently undertaken, at increasing cost, a collection of serialized messages represents all the versions of a given schema that ever existed; this imposes a larger backwards compatibility burden on data consumers, so more care is needed when evolving the schema.

## Part 2: Some basics

### Actually use the types
IDLs like Thrift, Protocol Buffers, and Avro have built-in type systems. Use them. Do not create a Map<String, String> and shove all your data in there; this negates the main reasons to use these IDLs in the first place. Think about your fields and use them appropriately.

### Use Enums
When a string field has an expected set of values, it is best to explicitly enumerate them via an enum construct, in order to get free sanity-checking from generated producer libraries, and to get easy space savings when using binary serialization.

### Field Naming
Make names as clear as possible. Time-related long fields tend to be ones that cause most confusion -- include the units in the field name (_ms for milliseconds,  _ns for nanoseconds, and so on). See also advanced modeling patterns for a more type-

### Stay Simple
Don’t over-complicate things if you can avoid it; flat data structures are easy to analyze and work with. Just because you can have nested structs, maps, and lists, doesn’t mean you should always use them. Be conservative with your schema complexity; the cost of understanding what you created will be paid by every consumer of the data for as long as your data sticks around (which will likely be much longer than you anticipated).

A completely flat structure can be efficiently processed by tools like Presto, Hive, Apache Impala, Amazon Redshift, Apache Spark, and others. Some of these tools can also process nested data, but it is likely they will not be as efficient or straightforward to use with nested data as with flat data.

Advanced techniques for handling schema evolution should be only used when advanced techniques are justified.

### Default values

Do not rely on special default values set via the IDL if you do not have (or expect not to have in the future) extremely tight control over all potential readers and writers. Unfortunately handling of missing values and application of defaults is not consistent across languages for some of these frameworks, and such issues are very tough to debug.

You don't want to find yourself reasoning through the difference between a value that happens to be equal to the default value and was serialized, and a default value that was generated for you by the reader because the field was not present in the serialized object. 

## Part 3: Schema Evolution

Schema evolution refers to changes one introduces to the schema over time. Changes one might want to introduce include:

* adding a field
* removing a field
* renaming a field
* changing a field's type
* making an optional field required
* making a required field optional
* adding or changing a field's default value

When you are dealing with the inertia of historical data, evolving schemas is hard because backwards compatibility has to be maintained, or data becomes unreadable.

It is also very useful to have your schema be as tight as possible, enforcing assumptions about data at write time and providing data producers with immediate feedback, and making it possible to easily prevent data problems via automated testing.

One approach commonly recommended when dealing with such IDLs amounts to the "optionals-only" pattern, laid out below. Major refactoring can be performed using another, more advanced, "union pattern". The Union pattern has to be committed to when you first design the schema, so don't skip reading that section of this document.

### Optionals-only pattern for minor schema changes

This approach ensures that old data remains valid, and new data can be (lossily) read by old readers. This approach does introduce another problem -- over time, baggage in the form of deprecated fields accumulates; one has to know not to trust some fields, and use others instead; a consumer who looks at the schema has a hard time understanding how things are supposed to fit together, and what can and cannot be relied on to be present. Comments only go so far to explain what's going on.

#### Optional vs required fields
Never use required fields because you might decide in the future they are not required. Adding new required fields renders old data invalid. A required field that's backwards-compatible via a default value is just an optional field. 

#### Adding Fields
Only ever add fields. Never delete fields. If you need to delete a field, just stop using it, and rename it to something like deprecated_foo_ms.

Thrift (possibly only older versions of thrift java libraries) ran into problems when an old reader ran into a new enum value -- pre-seed with “reserved_enum_value_1” and so on, always keep a bunch at the end.

#### Renaming Fields
Renaming a field is dangerous. There is likely to be code out there getting fields by name, not by id (especially if you serialize as JSON). Avoid the temptation.
The only exception is the above advice to rename deprecated fields. This is fine since no one should actually be using the field at this point to write new data; old readers not knowing about this new name won’t be a problem, since there is nothing for them to fail to read because of this.

#### Never, ever, change types
If you need to change a field’s type, add a new field and deprecate the old one. Also, consider just not skimping on bits. The majority of type migrations we’ve seen have been ints to doubles.
Once it’s in master, it’s in use
If you are part of a large technical organization that uses such IDLs for data serialization widely, it is rare to be able to guarantee that something on the master branch has not actually been used. It’s safest to assume that once a patch hits master, it’s production and all of the above schema evolution rules apply.

### Union pattern for major refactoring

The Union pattern that allows significant refactoring to be introduced without sacrificing the ability to read old data.

Only being able to add fields can eventually result in pretty ugly schemas, with a ton of deprecated fields one must know to ignore. The Union pattern was identified as a way to allow major schema refactoring to take place, while allowing code that knows about the new schema to still do a reasonable job with old data.

Essentially, the idea is to wrap your actual schema in a Union of all major versions of the schema:
```
Struct foo_struct_v1 {
  apache_timestamp: string,
  user_id: int,
  resource: string
}

Struct foo_struct_v2 {
   epoch_time_ms: long,
   user_id: long,
   resource: string,
   url_params: Map<string, string>
}

Union foo {
  foo_v1: foo_struct_v1,
  foo_v2: foo_struct_v2
}
```

In this example, foo is the actual message being produced and consumed. When major changes to the schema are introduced, such as going from a String representation of time to a number of milliseconds since the Linux epoch, or the user_id space is bumped up to longs, a new version of the struct is introduced.

Reader code can now be augmented with a converter from the old struct to the new struct, using custom logic for the conversion (eg, doing parsing on the fly for the old records, but not needing to do so for new ones). This converter can be inserted into a generally shared consumer library, so that individual programs do not need to concern themselves with handling old fields.

With a little bit of cleverness, one can build an automated cascade of converters such that a consumer can seamlessly consume all historical data that comes in N different schemas, with having only converters from an ith schema to i+1st schema.

This pattern does assume you have reasonable control over consuming libraries. It allows for safe major refactoring of schemas, allowing one to perform major migrations such as moving from parsed strings for dates into numerical timestamps, or other, more complex calculations. It also frees one from having to migrate the old data, assuming that read-time conversion is an acceptable cost.

Note also that this approach requires that all consumers be upgraded to a new schema before producers are. If an old reader encounters a new schema, it will fail to parse the new data. This is arguably preferable to interpreting the data by using old definitions.

If you adopt this pattern, it is still advisable to be conservative when introducing new schema versions -- only use it when the changes are really not backwards compatible. The "add optional fields" approach can be used for small incremental changes.

Note that this method does introduce more complexity than a simple struct carries. Use good judgement.

## Other Topics

### Advanced Types
Timestamp types and the like. Automated validation,
Scala has fancy things you can hang onto specialized types, so decorating an int64 as a “TimestampMillis” type can have real benefits in that kind of environment.

### Maps, features, and debug info
Yes but many caveats about when this is appropriate and how to stay out of the abyss [to be filled in]

## Further Reading

[Confluent Blog: How I learned to Stop Worrying and Love the Schema](https://www.confluent.io/blog/how-i-learned-to-stop-worrying-and-love-the-schema-part-1/) by Geoff Anderson

[Confluent Blog: Yes, Virginia, You Really Do Need a Schema Registry](https://www.confluent.io/blog/schema-registry-kafka-stream-processing-yes-virginia-you-really-need-one/) by Gwen Shapira

[O'Reilly Radar: The problem of managing schemas](https://www.oreilly.com/ideas/the-problem-of-managing-schemas) by Gwen Shapira

[Apache Avro documentation: Schema resolution rules](https://avro.apache.org/docs/1.8.1/spec.html#Schema+Resolution) Avro's built-in schema resolution rules (which may not always be sufficient for major refactoring, which is why you need all of the above)
