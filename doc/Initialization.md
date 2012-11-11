---
layout : doc
title : Initialization
group : doc
comments : true
---
SORM API was designed to avoid scatterring accross multiple components. It is centralized in a single class [`Instance`](/api/#sorm.Instance) - you use this class both to access the database and to initialize SORM.

There aren't many things you'll need to configure in SORM, and most of them are quite obvious, so first off we'll start with a complete example of SORM initialization.

## Example
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

## Entities
As you already know SORM works with data models designed with case classes, which are also called entities. In order for SORM to generate a correct database schema and to know how to work with your model generally, you need to register all entities of your model with it. You do that with a help of a [`sorm.Entity`](/api/#sorm.Entity$) case class, in a type-argument of which you specify the case class and in optional arguments `unique` and `indexed` you specify the keys.

## Keys
### Types of keys
There are two types of keys in SORM:

* Unique - a sequence of names of entity properties which together form a unique value. When possible this one gets mapped to a standard sql `UNIQUE KEY`.

* Index - a sequence of names of entity properties which are planned to be used together in queries which require optimization. By "being used" it is implied that they will get specified in either a "where" or an "order"-clause of a query. Usually this one maps to a standard sql `INDEX`.

### Specifying

{% highlight scala %}
Entity[Task](unique = Set() + Seq("name")),
             indexed = Set() + Seq("started") + Seq("closed", "outdated", "started"))
{% endhighlight %}
You can see a declaration of a unique key for a single property "name" and two indexes - for a single property "started" and for a group of properties "closed", "outdated" and "started".

### What happens if a property is impossible to map to a key on the DB side?

Nothing happens. The key just doesn't get set, but no exceptions get thrown. Your application remains in a perfectly working state, but just not optimized for that specific key. The same happens when the DB adapter doesn't support a key.

## Pool size
SORM is designed to support connection pooling out of the box. This allows your multithreaded application to effectively utilize multiple connections to your datasource. 

This feature becomes essential when your application puts a 100% load on a db connection. Since a db connection is executed on a single thread it will only let you utilize the power of a single processor at max, therefor for such applications multiple connections are essential. Luckily SORM doesn't just provide you with the ability to have multiple connections, it also automatically manages them for you. All you need to do is just specify how many connections you want to be opened at max with the `poolSize` parameter.

## Initialization mode
Each time you launch your application SORM performs initialization. For purposes of running your application in different environments (dev, production) multiple initialization modes were implemented. You specify them with an `initMode` parameter.
* [`DropAllCreate`](/api/#sorm.InitMode$) - wipe out all the contents of the db and generate the new tables. Useful for development. Complete data loss risks - be very careful not to specify this option in production environment
* [`DropCreate`](/api/#sorm.InitMode$) - drop only the tables which have conflicting names with the ones to be generated and generate the new ones. Useful for development when there are some tables unrelated to SORM present in the same database. Complete data loss risks - be very careful not to specify this option in production environment
* [`Create`](/api/#sorm.InitMode$) - try to generate the tables if they don't already exist. No dataloss risks, but when you change your domain, incompatibility errors may arise. Safe to use in production.
* [`DoNothing`](/api/#sorm.InitMode$) - no changes get applied to schema. Safe to use in production.

