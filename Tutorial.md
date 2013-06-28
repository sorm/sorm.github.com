---
title: Tutorial
group: navigation
layout: page
comments: true
keywords:
  - sorm tutorial
  - tutorial
---

<style>
  blockquote p { font-size: smaller; }
</style>

In this tutorial it is assumed that you know how to set up Maven or SBT projects.

##Let's add dependencies to our sample project
A Maven dependency to SORM looks like so: 

{% highlight xml %}
<dependency>
  <groupId>org.sorm-framework</groupId>
  <artifactId>sorm</artifactId>
  <version>0.3.8</version>
</dependency>
{% endhighlight %}

For our testing project we will use an in-memory version of H2 database - let's add a dependency for it too:

{% highlight xml %}
<dependency>
  <groupId>com.h2database</groupId>
  <artifactId>h2</artifactId>
  <version>1.3.168</version>
</dependency>
{% endhighlight %}

##Now, to the actual program
Consider a data model described with these standard case classes:

{% highlight scala %}
case class Artist
  ( names : Map[Locale, Seq[String]],
    genres : Set[Genre] )

case class Genre
  ( names : Map[Locale, Seq[String]] )

case class Locale
  ( code : String )
{% endhighlight %}

You can see that instead of assigning the `Genre` and `Artist` objects with a common `name` property we've decided here to go a bit tougher way to let our application support different locales and to allow it to store different variants of names for the same entity if there exist alternatives. For instance, in the English locale one artist we will store will have three alternative names: "The Rolling Stones", "Rolling Stones" and "Rolling Stones, The".

Now, for a minute, just imagine what it would take to represent such a model in a relational DB...

##Let's initialize SORM:

{% highlight scala %}
import sorm._

object Db extends Instance(
  entities = Set(
    Entity[Artist](),
    Entity[Genre](),
    Entity[Locale](unique = Set() + Seq("code"))
  ),
  url = "jdbc:h2:mem:test",
  user = "",
  password = "",
  initMode = InitMode.Create
)
{% endhighlight %}

With the code above we've created a SORM instance ready to work with a set of object types `Artist`, `Genre` and `Locale`. We've specified that SORM should create an appropriate unique key for a `Locale` property `code`. That instance will connect to an in-memory H2 database without specifying user or password and will generate schema tables if they don't exist already.

Guess what, that's it! As soon as you refer to this `Db` object you'll get an up and running database with a full schema structure generated automatically. All that's left to do is put it to use. 

##Let's populate the db with some data:

{% highlight scala %}
//  create locales:
val ru = Db.save( Locale("ru") )
val en = Db.save( Locale("en") )

//  create genres:
val rock      = Db.save( Genre( Map( en -> Seq("Rock"),
                                     ru -> Seq("Рок") ) ) )
val hardRock  = Db.save( Genre( Map( en -> Seq("Hard Rock"),
                                     ru -> Seq("Тяжёлый рок", 
                                               "Тяжелый рок") ) ) )
val metal     = Db.save( Genre( Map( en -> Seq("Metal"),
                                     ru -> Seq("Метал") ) ) )
val grunge    = Db.save( Genre( Map( en -> Seq("Grunge"),
                                     ru -> Seq("Грандж") ) ) )

//  create artists:
Db.save( Artist( Map( en -> Seq("Metallica"),
                      ru -> Seq("Металика", "Металлика") ),
                 Set( metal, rock, hardRock ) ) )
Db.save( Artist( Map( en -> Seq("Nirvana"),
                      ru -> Seq("Нирвана") ),
                 Set( rock, hardRock, grunge ) ) )
Db.save( Artist( Map( en -> Seq("Kino"),
                      ru -> Seq("Кино") ),
                 Set( rock ) ) )
Db.save( Artist( Map( en -> Seq("The Rolling Stones",
                                "Rolling Stones",
                                "Rolling Stones, The"),
                      ru -> Seq("Ролинг Стоунз",
                                "Роллинг Стоунз",
                                "Роллинг Стоунс",
                                "Ролинг Стоунс") ),
                 Set( rock ) ) )
{% endhighlight %}


##Now let's fetch some data from the populated database:

{% highlight scala %}
//  An artist by id `2`.
//  The result type is `Option[Artist with Persisted]`
val nirvana = Db.query[Artist].whereEqual("id", 2).fetchOne() 
//  Please note that for HSQLDB instead of "Nirvana" it will be "Kino", since
//  in that db indexes are zero-based

//  All artists having a genre equaling to the value of the `metal` variable, 
//  which we've previously declared. 
//  The result type is `Stream[Artist with Persisted]`
val metalArtists = Db.query[Artist].whereContains("genres", metal).fetch()

//  All artists having a genre that contains "Hard Rock" of a locale with a 
//  code "en" in a list of its names.
//  The result type is `Stream[Artist with Persisted]`
val hardRockArtists 
  = Db.query[Artist]
      .whereEqual("genres.item.names.value.item", "Hard Rock")
      .whereEqual("genres.item.names.key.code", "en")
      .fetch()
{% endhighlight %}
