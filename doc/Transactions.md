---
layout : doc
title : Transactions
group : doc
comments : true
---

Transactions allow to perform multiple db-requests in such a way that when any failure occurs all the changes that were made with this transaction get cancelled. For most dbs transactions also provide guarantees that nothing will get changed in between the db-requests in multithreaded applications.

All db-requests which should be executed as part of transaction must be run on the same thread.

Use transactions with care because for the time the transaction is being executed the involved tables are supposed to get locked, putting all the requests to them from other threads in a queue until the current transaction finishes. The best practice is to make transactions as short as possible and to perform calculations prior to entering a transaction.

###Example #1
{% highlight scala %}
Db.transaction {
  Db.save(account.copy(balance = account.balance - order.price))
  Db.save(order.copy(paid = true))
}
{% endhighlight %}

This example shows a classical situation when transaction is essential. It ensures that either both save operations succeed or none of them gets performed, which guarantees that if any kind of failure happens it won't happen so that the account will get a balance reduced without the order being marked as paid. 

###Example #2
{% highlight scala %}
val task : Option[Task] 
  = Db.transaction {
      Db.query[Task].whereEqual("busy", false).fetchOne()
        .map(_.copy(busy = true))
        .map(Db.save)
    }
{% endhighlight %}

In this example transaction is used to ensure that the same task won't be fetched on multiple threads.