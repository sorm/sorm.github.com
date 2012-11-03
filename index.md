---
layout: page
title: SORM
tagline: A simple and elegant way to do persistance
description: An index page
---


SORM is an object-relational mapping framework having consistency and simplicity as its primary principles. It is absolutely abstracted from relational side, automagically creating database tables, emitting queries, inserting, updating and deleting records. This all functionality is presented to the user with a simple API around standard Scala's case classes.

##A Quick Glance

Declare a model:

{% highlight scala %}
case class Artist ( name : String, genres : Set[Genre] )
case class Genre ( name : String ) 
{% endhighlight %}

Create a SORM instance, which automatically generates the schema:

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

##Installation and Latest Info
For installation instructions and latest release information please visit the project on [GitHub](https://github.com/nikita-volkov/sorm).

##Learn more
You can see how SORM compares to the currently dominant Scala ORM Framework [here](/pages/SORM-vs-Slick.html) or continue on to a more comprehensive [Tutorial](/pages/Tutorial.html). For detailed documentation please visit the [Wiki](https://github.com/nikita-volkov/sorm/wiki) or learn the [API](/api/) (you're really interested in the contents of a plain `sorm._` package only).

##Support
Support is provided at [StackOverflow](http://stackoverflow.com/questions/tagged/sorm). Go ahead and ask your questions under a tag "sorm".

##Issues
Please, post any issues you come across in the issues section.

##Contribution
It is a very large project, and any kind of contribution is much appreciated. So if you find anything that you think SORM could evolve on, go ahead and [fork it](https://github.com/nikita-volkov/sorm)! Currently, the most wanted contributions are drivers for other DBRMs.