---
title: Tutorial
group: navigation
layout: page
---

###Let's add dependencies to our sample project
A Maven dependency to SORM looks like so: 

{% highlight xml %}
<dependency>
  <groupId>com.github.nikita-volkov</groupId>
  <artifactId>sorm</artifactId>
  <version>0.2.0</version>
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

###Now, to the actual program
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

> You can see that instead of assigning the `Genre` and `Artist` objects with a common `name` property we've decided here to go a bit tougher way to let our application support different locales and to allow it to store different variants of names for the same entity if there exist alternatives. For instance, in the English locale one artist we will store will have three alternative names: "The Rolling Stones", "Rolling Stones" and "Rolling Stones, The".

###Let's initialize our SORM instance:

{% highlight scala %}
import sorm._

val db
  = new Instance(
      entities = Set() + Entity[Artist]() + Entity[Genre]() + Entity[Locale](),
      url = "jdbc:h2:mem:test",
      user = "",
      password = "",
      initMode = InitMode.Create
    )
{% endhighlight %}

> If you need an explanation with the code above, we create a SORM instance ready to work with objects of types `Artist`, `Genre` and `Locale`. That instance connects to an in-memory H2 database without specifying user or password and creates the schema tables if they don't exist already.

Guess what, that's it! We now have an up and running database with a full schema structure required for storing our objects already created for us. All that's left to do is put it to use. 

###Let's open a connection to that db:

{% highlight scala %}
val cx = db.connection()
{% endhighlight %}

###Let's populate it with some data:

{% highlight scala %}
//  create locales:
val ru = cx.save( Locale("ru") )
val en = cx.save( Locale("en") )

//  create genres:
val rock      = cx.save( Genre( Map( en -> Seq("Rock"),
                                     ru -> Seq("Рок") ) ) )
val hardRock  = cx.save( Genre( Map( en -> Seq("Hard Rock"),
                                     ru -> Seq("Тяжёлый рок", 
                                               "Тяжелый рок") ) ) )
val metal     = cx.save( Genre( Map( en -> Seq("Metal"),
                                     ru -> Seq("Метал") ) ) )
val grunge    = cx.save( Genre( Map( en -> Seq("Grunge"),
                                     ru -> Seq("Грандж") ) ) )

//  create artists:
cx.save( Artist( Map( en -> Seq("Metallica"),
                      ru -> Seq("Металика", "Металлика") ),
                 Set( metal, rock, hardRock ) ) )
cx.save( Artist( Map( en -> Seq("Nirvana"),
                      ru -> Seq("Нирвана") ),
                 Set( rock, hardRock, grunge ) ) )
cx.save( Artist( Map( en -> Seq("Kino"),
                      ru -> Seq("Кино") ),
                 Set( rock ) ) )
cx.save( Artist( Map( en -> Seq("The Rolling Stones",
                                "Rolling Stones",
                                "Rolling Stones, The"),
                      ru -> Seq("Ролинг Стоунз",
                                "Роллинг Стоунз",
                                "Роллинг Стоунс",
                                "Ролинг Стоунс") ),
                 Set( rock ) ) )
cx.save( Artist( Map( en -> Seq("Dire Straits"),
                      ru -> Seq("Даэр Стрэйтс") ),
                 Set( rock ) ) )
cx.save( Artist( Map( en -> Seq("Godsmack"),
                      ru -> Seq("Годсмэк") ),
                 Set( metal, hardRock, rock ) ) )
{% endhighlight %}


###Now let's fetch some data from our populated database:

{% highlight scala %}
//  get an artist by id:
val nirvana
  = cx.access[Artist].whereEquals("id", 2).fetchOne() // will return Nirvana

//  all artists having a style that has `Hard Rock` in a list of its names
val hardRockArtists  = cx.access[Artist].whereEquals("names.value.item.value", "Hard Rock").fetch()
{% endhighlight %}
