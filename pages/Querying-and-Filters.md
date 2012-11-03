
All the querying functionality is provided through the `access[T]` method of `sorm.Connection`. The principle is simple: by calling `access[T]` you create a query, then modify it by calling different methods on it, which return copies of this query with appropriate modifications. After the last modification method you call one of the fetching methods on it, which actually does emit the query and fetches the results from the db.

## Property path
All modifier methods of [`Access`](http://nikita-volkov.github.com/sorm/api/#sorm.Access) accept a `String` value as first parameter. This value specifies a path to a symbol in a tree of the accessed object. This path is specified as a dot-delimited nodes. 

For example, a path value `"genres.item.name"` can refer to a property `name` of an object stored in a `Seq` which is a value of a `genres` property of the entity being accessed.

Following is a reference on what the nodes can be.
* Property. A name of a property
* Tuple item index. Same as in Scala: `_3` will refer to a third item of a tuple
* Range `start` and `end`
* Option, Seq, Set `item`
* Map `key` and `value`

## The `or` conditions
Stacking filters on top of each other with piped "where"-clauses produces an `and` logic, to introduce `or` conditions to your query you'll have to use the DSL and a special `where` method of [`Access`](http://nikita-volkov.github.com/sorm/api/#sorm.Access).

Here's how you'd do a query to select an artist who has a genre equaling either "metal" or "rock", but excluding "Metallica":

    import sorm.FilterDsl._

    cx.access[Artist]
      .where( 
        ( ( "genre" equal "metal" ) or
          ( "genre" equal "rock" ) ) and
        ( "name" notEqual "Metallica" )
      )
      .fetchOne()

> The above implies `case class Artist ( name : String, genre : String )`