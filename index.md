---
layout: page
title: Introduction
header: SORM
tagline: An elegant and scalable way to do persistence in Scala 
comments: false
---

SORM is a purely Scala-oriented object-relational mapping framework designed to eliminate boilerplate and maximize productivity.

##Features

* Complete abstraction from relational concepts. You work with case classes, collections and other standard Scala data types instead of tables, rows, foreign keys and one-to-many relations
* No SQL. Just a simple querying and modification API working with case classes
* A simple, intuitive and centralized yet powerful API of just a few methods. No tangled implicit constructions and functionality scatterred accross multiple components
* Decoupled modelling. You design a data model without any annotations or data-types inflicted by SORM - just standard Scala case classes with standard Scala data types 
* Support for most standard Scala data types, including `Seq`, `Set`, `Map`, `Option`, `Tuple`, `Enumeration` and all primitives
* Automated schema generation
* Multithreading. A single SORM instance can safely be used accross multiple threads
* Integrated connections pooling. Scalable just by setting a single parameter - `poolSize`
* Connection-agnostic API. All connection-management-related tasks are abstracted away from. You work with a single SORM instance and it takes care itself of whether and when to open a new connection, pool it or close it 

##A Quick Intro

This is a complete example. No extra code is required for it to work.

Declare a model:

{% highlight scala %}
case class Artist ( name : String, genres : Set[Genre] )
case class Genre ( name : String ) 
{% endhighlight %}

Create a SORM instance, which automatically generates the schema:

{% highlight scala %}
import sorm._
object Db extends Instance (
  entities = Set() + Entity[Artist]() + Entity[Genre](),
  url = "jdbc:h2:mem:test"
)
{% endhighlight %}

Store values in the db:

{% highlight scala %}
val metal = Db.save( Genre("Metal") )
val rock = Db.save( Genre("Rock") )
Db.save( Artist("Metallica", Set() + metal + rock) )
Db.save( Artist("Dire Straits", Set() + rock) )
{% endhighlight %}

Retrieve values from the db:

{% highlight scala %}
val metallica = Db.query[Artist].whereEqual("name", "Metallica").fetchOne() // Option[Artist]
val rockArtists = Db.query[Artist].whereEqual("genres.name", "Rock").fetch() // Stream[Artist]
{% endhighlight %}

##Installation and Latest Release Info
Latest release info and installation instructions are provided at the project's [GitHub page](https://github.com/nikita-volkov/sorm#readme).

##Learn More
You can see how SORM compares to the currently dominant Scala ORM Framework [here](/SORM-vs-Slick.html) or continue on to a more comprehensive [Tutorial](/Tutorial.html). For detailed documentation please visit [this page](/Documentation.html) or learn the [API](/api/) (you're really interested in the contents of a plain `sorm._` package only).

##Support
Support is provided at [StackOverflow](http://stackoverflow.com/questions/tagged/sorm). Go ahead and ask your questions under a tag "sorm".

##Issues
Please, post any issues you come across [here](https://github.com/nikita-volkov/sorm/issues).

##Contribution
Any kind of contribution is much appreciated. If you find anything that you think SORM could evolve on, go ahead and [fork it](https://github.com/nikita-volkov/sorm)! 