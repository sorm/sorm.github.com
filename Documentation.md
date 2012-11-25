---
layout: with_navigation
title: Documentation
group: navigation
comments: true
navigation: 
  - name: Initialization
    anchor: initialization
  - name: Persisted trait and ids
    anchor: persisted_trait_and_ids
  - name: Saving
    anchor: saving
  - name: Querying
    anchor: querying
  - name: Transactions
    anchor: transactions
  - name: Data Types
    anchor: data_types
  - name: Tracing SQL
    anchor: tracing_sql
keywords:
  - SORM Documentation
  - Documentation
  - SORM Reference
  - Reference
---
<style>

  hr { margin: 70px 14px 70px 14px !important; }
  
</style>


## Initialization

SORM API was designed to avoid scatterring accross multiple components. It is centralized in a single class [`Instance`](/api/#sorm.Instance) - you use this class both to access the database and to initialize SORM.

There aren't many things you'll need to configure in SORM, and most of them are quite obvious, so first off we'll start with a complete example of SORM initialization.

### Example
{% highlight scala %}
case class Artist ( name : String, genres : Set[Genre] )
case class Genre ( name : String ) 

import sorm._

object Db extends Instance (
  entities = Set() + Entity[Artist]() + Entity[Genre](unique = Set() + Seq("name")),
  url = "jdbc:h2:mem:test",
  user = "",
  password = "",
  initMode = InitMode.Create,
  poolSize = 12
)
{% endhighlight %}
In the code above you can see a data model declaration, which is followed by SORM instance configured to work with case classes `Artist` and `Genre`. This instance will create an appropriate unique key for a `Genre` property `name`. It will connect to an in-memory H2 database named `test` without specifying user or password and will generate schema tables if they don’t already exist. It will also manage a pool of 12 connections, allowing you to utilize as many to connect to your db parallelly.

### Entities
As you already know SORM works with data models designed with case classes, which are also called entities. In order for SORM to generate a correct database schema and to know how to work with your model generally, you need to register all entities of your model with it. You do that with a help of a [`sorm.Entity`](/api/#sorm.Entity$) case class, in a type-argument of which you specify the case class and in optional arguments `unique` and `indexed` you specify the keys.

### Keys
#### Types of keys
There are two types of keys in SORM:

* Unique - a sequence of names of entity properties which together form a unique value. When possible this one gets mapped to a standard sql `UNIQUE KEY`.

* Index - a sequence of names of entity properties which are planned to be used together in queries which require optimization. By "being used" it is implied that they will get specified in either a "where" or an "order"-clause of a query. Usually this one maps to a standard sql `INDEX`.

#### Specifying

{% highlight scala %}
Entity[Task](unique = Set() + Seq("name")),
             indexed = Set() + Seq("started") + Seq("closed", "outdated", "started"))
{% endhighlight %}
You can see a declaration of a unique key for a single property "name" and two indexes - for a single property "started" and for a group of properties "closed", "outdated" and "started".

#### What happens if a property is impossible to map to a key on the DB side?

Nothing happens. The key just doesn't get set, but no exceptions get thrown. Your application remains in a perfectly working state, but just not optimized for that specific key. The same happens when the DB adapter doesn't support a key.

### Pool size
SORM is designed to support connection pooling out of the box. This allows your multithreaded application to effectively utilize multiple connections to your datasource. 

This feature becomes essential when your application puts a 100% load on a db connection. Since a db connection is executed on a single thread it will only let you utilize the power of a single processor at max, therefor for such applications multiple connections are essential. Luckily SORM doesn't just provide you with the ability to have multiple connections, it also automatically manages them for you. All you need to do is just specify how many connections you want to be opened at max with the `poolSize` parameter.

### Initialization mode
Each time you launch your application SORM performs initialization. For purposes of running your application in different environments (dev, production) multiple initialization modes were implemented. You specify them with an `initMode` parameter.
* [`DropAllCreate`](/api/#sorm.InitMode$) - wipe out all the contents of the db and generate the new tables. Useful for development. Complete data loss risks - be very careful not to specify this option in production environment
* [`DropCreate`](/api/#sorm.InitMode$) - drop only the tables which have conflicting names with the ones to be generated and generate the new ones. Useful for development when there are some tables unrelated to SORM present in the same database. Complete data loss risks - be very careful not to specify this option in production environment
* [`Create`](/api/#sorm.InitMode$) - try to generate the tables if they don't already exist. No dataloss risks, but when you change your domain, incompatibility errors may arise. Safe to use in production.
* [`DoNothing`](/api/#sorm.InitMode$) - no changes get applied to schema. Safe to use in production.




---

## Persisted trait and ids

All the entities returned from SORM have a [`Persisted`](/api/#sorm.Persisted) trait with an appropriate value of `id` mixed in. This is what lets SORM decide whether to `INSERT` rows or `UPDATE` them (and which ones) when the `save` operation is called. This also provides you with access to its generated `id`.

Since the `id` property value is meant to be generated by database, it is protected from the user of being able to manually specify it as well as letting the case classes have such a property.

So, instead of 
    
{% highlight scala %}
case class Artist ( id : Long, name : String ) 
{% endhighlight %}

you should use 

{% highlight scala %}
case class Artist ( name : String ) 
{% endhighlight %}

and let SORM take care of `id` property management for you.

### What should you do when you need to get an id of an entity?

Just do

{% highlight scala %}
artist.id 
{% endhighlight %}

but for you to be able to do that the `artist` value must have a `Persisted` trait mixed in (i.e., have a type `Artist with Persisted`), which can happen only in three cases:

1. When you store a value in the db: {% highlight scala %}
val artist = Db.save(Artist("Metallica")) 
{% endhighlight %}
    
2. When you fetch it from the db: {% highlight scala %}
val artist = Db.query[Artist].whereEqual("name", "Metallica").fetchOne().get
{% endhighlight %}

3. When you make a copy of an already persisted entity: {% highlight scala %}
val artist = someOtherPersistedArtist.copy(name = "METALLICA")
{% endhighlight %}

### What should you do when you need an entity by id?

Either do a standard fetch, which will return an `Option`:
{% highlight scala %}
Db.query[Artist].whereEqual("id", 234).fetchOne()
{% endhighlight %}
or use a special method `fetchById`, which will either return the appropriate entity or fail if it doesn't exist:
{% highlight scala %}
Db.fetchById[Artist](234)
{% endhighlight %}

### What should you do when you need to specify the id yourself?

Nothing. You shouldn't do that. The `id` property is supposed to be generated and managed by db only. If you need to specify some external unique identifier, like, for instance, Amazon's ASIN, just add an appropriate field to your entity and specify it as unique on SORM instantiation. 

---

## Saving

Let's assume you have a following model registered with SORM `Instance`:

{% highlight scala %}
case class Genre ( name : String )
case class Artist ( name : String, genres : Set[Genre] )
{% endhighlight %}

### Storing a new entity

{% highlight scala %}
val metal : Genre with Persisted
  = Db.save(Genre("Metal"))
val metallica : Artist with Persisted
  = Db.save(Artist("Metallica", Set(metal))) 
{% endhighlight %}

Please note that executing `Db.save(Artist("Metallica", Set(Genre("Metal"))))` right away is prohibited, since the entities you want to persist may refer only to already persisted ones, and a freshly created `Genre("Metal")` as in this case is not yet persisted.

### Updating a persisted entity

{% highlight scala %}
val rock : Genre with Persisted
  = Db.save(Genre("Rock"))

val metallica : Option[Artist with Persisted]
  = Db.query[Artist]
      .whereEqual("name", "Metallica")
      .fetchOne()   //  fetch an entity from db
                    //  (returns `Option[Artist with Persisted]`)
      .map(a => a.copy(genres = a.genres + rock))
                    //  update the entity
      .map(Db.save) //  persist the updates to entity
{% endhighlight %}

Please note that the types are specified only for reference.




---

## Querying

All the querying functionality is provided through the `query[T]` method of a SORM instance. The principle is simple: by calling `query[T]` you create an immutable [`Querier`](/api/#sorm.Querier) object, then stack different "modifier" methods on it, which return copies of this object with appropriate modifications - very similar to "Builder" pattern, just the functional way. After the last modification method you call one of the fetching methods on it, which actually does emit the query and fetches the results from the db.

{% highlight scala %}
val artists
  = Db.query[Artist]
      .whereNotContains("genre", pop)
      .limit(3) // The order of "modifier methods" doesn't matter much
      .orderBy("name")
      .whereNotContains("genre", rock) // Stacking "where" clauses produces the "and" logic
      .fetch() // the sql query gets emitted only at this point
{% endhighlight %}

With the query above we've fetched three artists which have neither pop or rock in the set of its genres. The results are ordered by artist name.


### Property path

All modifier methods of [`Querier`](http://nikita-volkov.github.com/sorm/api/#sorm.Querier) accept a `String` value as a first parameter. This value specifies a path to a symbol in a tree of the accessed entity. Path is specified as dot-delimited nodes. 

#### Path tree nodes reference

* Property. A name of a property
* Tuple item index. Same as in Scala: `_3` will refer to a third item of a tuple
* Range `start` and `end`
* Option, Seq, Set `item`
* Map `key` and `value`

#### Example path tree

Here's a decomposed tree of an `Artist` object taken from the [Tutorial](/Tutorial.html) with types explained:
    
      Path Tree         |   Type Tree
    --------------------+----------------------------------
      *                 |   Artist
      - names           |   - Map[Locale, Seq[String]]
      | - key           |   | - Locale
      | | - code        |   | | - String
      | - value         |   | - Seq[String]
      |   - item        |   |   - String
      - genres          |   - Set[Genre]
        - item          |     - Genre
          - names       |       - Map[Locale, Seq[String]]
            - key       |         - Locale
            | - code    |         | - String
            - value     |         - Seq[String]
              - item    |           - String

Here's the assumed model:
{% highlight scala %}
case class Artist ( names : Map[Locale, Seq[String]], genres : Set[Genre] )
case class Genre ( names : Map[Locale, Seq[String]] )
case class Locale ( code : String )
{% endhighlight %}

Keeping the above table in mind here's how we can query for artists having "pop" among the names of their genres:
{% highlight scala %}
Db.query[Artist].whereEqual("genres.item.names.value.item", "pop").fetch()
// or
Db.query[Artist].whereContains("genres.item.names.value", "pop").fetch()
{% endhighlight %}


### The "or" conditions and DSL

Stacking filters on top of each other with piped "where"-clauses produces an `and` logic, to introduce `or` conditions to your query you'll have to use the DSL and a special `where` method of [`Querier`](http://nikita-volkov.github.com/sorm/api/#sorm.Querier).

Here's how you'd do a query to select an artist who has a genre equaling either "metal" or "rock", but excluding "Metallica":

{% highlight scala %}
import sorm.Dsl._

Db.query[Artist]
  .where( 
    ( ( "genre" equal "metal" ) or
      ( "genre" equal "rock" ) ) and
    ( "name" notEqual "Metallica" )
  )
  .fetchOne()
{% endhighlight %}

The above implies `case class Artist ( name : String, genre : String )`




---

## Transactions

Transactions allow to perform multiple db-requests in such a way that when any failure occurs all the changes that were made with this transaction get cancelled. For most dbs transactions also provide guarantees that nothing will get changed in between the db-requests in multithreaded applications.

All db-requests which should be executed as part of transaction must be run on the same thread.

Use transactions with care because for the time the transaction is being executed the involved tables are supposed to get locked, putting all the requests to them from other threads in a queue until the current transaction finishes. The best practice is to make transactions as short as possible and to perform calculations prior to entering a transaction.

#### Example #1
{% highlight scala %}
Db.transaction {
  Db.save(account.copy(balance = account.balance - order.price))
  Db.save(order.copy(paid = true))
}
{% endhighlight %}

This example shows a classical situation when transaction is essential. It ensures that either both save operations succeed or none of them gets performed, which guarantees that if any kind of failure happens it won't happen so that the account will get a balance reduced without the order being marked as paid. 

#### Example #2
{% highlight scala %}
val task : Option[Task] 
  = Db.transaction {
      Db.query[Task].whereEqual("busy", false).fetchOne()
        .map(_.copy(busy = true))
        .map(Db.save)
    }
{% endhighlight %}

In this example transaction is used to ensure that the same task won't be fetched on multiple threads.




---

## Data Types

#### Case Classes
Naturally, you represent your entities with them.

#### All Standard Primitives
`Boolean`, `Byte`, `Char`, `Double`, `Float`, `Int`, `Long`, `Short`

#### String
Strings support has a wrapped logic: if a `String` property is specified as part of some kind of key (unique, index) on a relational side it is represented as a `VARCHAR` and thus is limited to have a maximum length of only 256 characters, otherwise it is represented as by a `CLOB` which is generally limited to ~4GB of data. I hope that's enough.

#### Option
Supported

#### Either
Not supported. Support may come in future releases. You can speed it up by voting for it on the [issue tracker](https://github.com/nikita-volkov/sorm/issues).

#### Collections
General types of immutable collections: `Seq`, `Set` and `Map` - are all supported. Please note that you should always use these general types instead of specific ones, i.e. `Seq` instead of `List`, `IndexedSeq` or `Vector` and etc.

#### Ranges
A simple range of form `1 to 21` is supported. The "until"-ranges or the ones specifying a step are unsupported.

#### Tuples
Supported

#### Enumerations
Standard Scala enumerations are supported

#### Joda Time
* `org.joda.time.DateTime` - to represent any given point in time of current era including timezone information with precision to milliseconds
* `org.joda.time.LocalDate` - to represent any date of current era without a timezone
* `org.joda.time.LocalTime` - to represent a time without a timezone

### So what do you get from this "support"?
You can seamlessly persist and query even the most ridiculously complex data structures, such as the following:

{% highlight scala %}
case class A ( map : Map[B, List[Set[(Int, String, Range)]]] ) 
case class B ( name : String )
{% endhighlight %}




---

## Tracing SQL

For debugging purposes it can be useful to see the SQL that SORM emits under the hood. For that you need to set its logging level to "debug". To do that you just need to add the following VM option to your app execution:

    org.slf4j.simplelogger.log.sorm=debug

Here's a complete sample execution command:

    java -Dorg.slf4j.simplelogger.log.sorm=debug -jar MyAmazingApp.jar
