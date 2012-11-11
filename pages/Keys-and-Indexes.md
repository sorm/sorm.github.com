---
title: Keys and Indexes
group: doc
layout: doc
comments: true
---

##Types of keys
In SORM there are two types of keys:

* Unique - a sequence of names of entity properties which together form a unique value. When possible this one maps to a standard sql `UNIQUE KEY`.

* Index - a sequence of names of entity properties which will be used together in queries which require optimization. By "being used" it is implied that they will get specified in either a "where" or an "order"-clause of a query. Usually this one maps to a standard sql `INDEX`.


##Specifying

Keys are specified on creation of a SORM instance as parameters to `Entity` instances: 
{% highlight scala %}
import sorm._
val db 
  = new Instance(
      entities
        = Set() +
          Entity[Tag](unique = Set() + Seq("name")) +
          Entity[Task](indexed = Set() + Seq("opened") + Seq("closed") + Seq("started") + Seq("outdated") + Seq("closed", "outdated", "started")),
        ...
    )
{% endhighlight %}

##What happens if the property is impossible to map to a key on the DB side?

Nothing happens. The key just doesn't get set, but no exceptions get thrown. Your application remains in a perfectly working state, but just not optimized for that specific key. The same happens when the DB adapter doesn't support a key.
