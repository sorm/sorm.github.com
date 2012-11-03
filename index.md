---
layout: page
title: SORM
tagline: A simple and elegant way to do persistance
description: An index page
---


SORM is an object-relational mapping framework having consistency and simplicity at its primary principles. It is absolutely abstracted from relational side, automagically creating database tables, emitting queries, inserting, updating and deleting records. This all functionality is presented to the user with a simple API around standard Scala's case classes.

##A Quick Glance

Declare a model:

{% highlight scala %}
case class Artist ( name : String, genres : Set[Genre] )
case class Genre ( name : String ) 
{% endhighlight %}

Create a sorm instance, which automatically generates the schema:

{% highlight scala %}
import sorm._
val db 
  = new Instance(
      entities = Set() + Entity[Artist]() + Entity[Style](),
      url = "jdbc:h2:mem:test",
      user = "",
      password = "",
      initMode = InitMode.Create
    )
{% endhighlight %}

Open a single connection:

{% highlight scala %}
val cx = db.connection()
{% endhighlight %}

Store values in a db:

{% highlight scala %}
val metal = cx.save( Genre("Metal") )
val rock = cx.save( Genre("Rock") )
cx.save( Artist("Metallica", Set() + metal + rock) )
cx.save( Artist("Dire Straits", Set() + rock) )
{% endhighlight %}

Retrieve a value from a db:

{% highlight scala %}
val metallica = cx.access[Artist].whereEqual("name", "Metallica").fetchOne()
{% endhighlight %}

##Diving Deeper
You can see how SORM compares to the currently dominant Scala ORM Framework [here](/pages/SORM-vs-Slick.html) or continue on to a [Tutorial](/pages/Tutorial.html).

##Installation and Latest Info
For installation instructions and latest release information please visit the project on [GitHub](https://github.com/nikita-volkov/sorm).
