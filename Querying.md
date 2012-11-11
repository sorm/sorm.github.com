---
layout : doc
title : Querying
group : doc
comments : true
---

All the querying functionality is provided through the `access[T]` method of a SORM instance. The principle is simple: by calling `access[T]` you create an `Access` object, then modify it by calling different methods on it, which return copies of this object with appropriate modifications - very similar to "Builder" pattern. After the last modification method you call one of the fetching methods on it, which actually does emit the query and fetches the results from the db.

## Property path

All modifier methods of [`Access`](http://nikita-volkov.github.com/sorm/api/#sorm.Access) accept a `String` value as a first parameter. This value specifies a path to a symbol in a tree of the accessed entity. This path is specified as a dot-delimited nodes. 

For example, a path value `"genres.item.name"` can refer to a property `name` of an entity stored in a `Seq` which is a value of a `genres` property of the entity being accessed.

Following is a reference on what the nodes can be.
* Property. A name of a property
* Tuple item index. Same as in Scala: `_3` will refer to a third item of a tuple
* Range `start` and `end`
* Option, Seq, Set `item`
* Map `key` and `value`

## The "or" conditions and DSL

Stacking filters on top of each other with piped "where"-clauses produces an `and` logic, to introduce `or` conditions to your query you'll have to use the DSL and a special `where` method of [`Access`](http://nikita-volkov.github.com/sorm/api/#sorm.Access).

Here's how you'd do a query to select an artist who has a genre equaling either "metal" or "rock", but excluding "Metallica":

{% highlight scala %}
import sorm.Dsl._

Db.access[Artist]
  .where( 
    ( ( "genre" equal "metal" ) or
      ( "genre" equal "rock" ) ) and
    ( "name" notEqual "Metallica" )
  )
  .fetchOne()
{% endhighlight %}

The above implies `case class Artist ( name : String, genre : String )`