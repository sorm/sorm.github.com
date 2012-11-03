Prior to reading this page please get yourself acquainted with the [Persisted entities identification concept](Persisted-Trait-and-Id) of SORM.

Let's assume you have a `SORM` connection assigned to `cx` variable and a following model registered with SORM `Instance`:

    case class Genre ( name : String )
    case class Artist ( name : String, genres : Set[Genre] )

## Storing a new entity
    
    val metal = cx.save(Genre("Metal")) // the value is of type `Genre with sorm.Persisted`
    val metallica = cx.save(Artist("Metallica", Set(metal))) 

> Please note that executing `cx.save(Artist("Metallica", Set(Genre("Metal"))))` right away is prohibited, since the entities you want to persist may refer only to already persisted ones, and a freshly created `Genre("Metal")` as in this case is not yet persisted.

## Updating a persisted entity
    
    val rock = cx.save(Genre("Rock"))

    val metallica : Option[Artist with Persisted]
      = cx.access[Artist]
          .whereEqual("name", "Metallica")
          .fetchOne()   //  fetch an entity from db
                        //  (returns `Option[Artist with sorm.Persisted]`)
          .map(a => a.copy(genres = a.genres + rock))
                        //  update the entity
          .map(cx.save) //  persist the updates to entity

