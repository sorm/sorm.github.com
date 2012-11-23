---
title: SORM vs. Slick
group: navigation
layout: page
comments: true
keywords: 
  - SORM vs Slick
alias:
  - /pages/SORM-vs-SLICK.html
---
<style>

  .yes { color: #A6E22E; }
  .no { color: #F92672; }


  .vs { margin-left: 0; margin-right: 0; }
  .vs > * { width: 50%; vertical-align: top; }

  /* positioning */
  .vs > * { display: inline-block; }

  /* gaps fix */
  .vs { font-size: 0; }
  .vs > * { font-size: 14px; }

  /* line */
  .vs > * { padding-left: 1%; border-left: solid thin; }
  .vs > :first-child { padding-right: 1%; padding-left: 0; border-left: none; }

  /* wrap fix */
  .vs { white-space: nowrap; }
  .vs > * { white-space: normal; }

  ul.vs { padding: 0; margin: 0; list-style: none; }

</style>

In the comparisons presented on this page the following data model will be assumed:

{% highlight scala %}
case class Coffee ( name : String, supplier : Supplier, price : Double, sales : Int, total : Int )
case class Supplier ( name : String, street : String, city : String, state : String, zip : String )
{% endhighlight %}


##Framework initialization

<ul class="vs">
  <li>
<h3>SORM</h3>
{% highlight scala %}
import sorm._

object Db extends Instance (
  entities = Set( Entity[Coffee](), 
                  Entity[Supplier]() ),
  url = "jdbc:h2:mem:test"
)
{% endhighlight %}

  </li>
  <li>
<h3>Slick</h3>
{% highlight scala %}
import scala.slick.driver.H2Driver.simple._
import Database.threadLocalSession

object Suppliers extends Table[(Int, String, String, String, String, String)]("SUPPLIERS") {
  def id = column[Int]("SUP_ID", O.PrimaryKey) 
  def name = column[String]("SUP_NAME")
  def street = column[String]("STREET")
  def city = column[String]("CITY")
  def state = column[String]("STATE")
  def zip = column[String]("ZIP")
  def * = id ~ name ~ street ~ city ~ state ~ zip
}

object Coffees extends Table[(String, Int, Double, Int, Int)]("COFFEES") {
  def name = column[String]("COF_NAME", 
                            O.PrimaryKey)
  def supID = column[Int]("SUP_ID")
  def price = column[Double]("PRICE")
  def sales = column[Int]("SALES")
  def total = column[Int]("TOTAL")
  def * = name ~ supID ~ price ~ sales ~ total
  def supplier = foreignKey("SUP_FK", supID, Suppliers)(_.id)
}

val db = Database.forURL("jdbc:h2:mem:test", driver = "org.h2.Driver")
{% endhighlight %}
  </li>
</ul>


##Connections management and pooling
<ul class="vs">
  <li>
      <h3>SORM</h3>
      <p>Automatic</p>
  </li>
  <li>
      <h3>Slick</h3>
      <p>Uses a single connection. Using 3rd-party connection poolers multiple connections can be achieved.</p>
  </li>
</ul>

##Database schema generation
<ul class="vs">
  <li>
      <h3>SORM</h3>
      <p>Automatic</p>
  </li>
  <li>
      <h3>Slick</h3>
{% highlight scala %}
db.withSession {
  (Suppliers.ddl ++ Coffees.ddl).create
}
{% endhighlight %}
  </li>
</ul>

##Inserting

Persist the following three objects:
{% highlight scala %}
val supplier1 = Supplier("Acme, Inc.", "99 Market Street", "Groundsville", "CA", "95199")
val supplier2 = Supplier("Superior Coffee", "1 Party Place", "Mendocino", "CA", "95460")
val supplier3 = Supplier("The High Ground", "100 Coffee Lane", "Meadows", "CA", "93966")
{% endhighlight %}

<ul class="vs">
  <li>
      <h3>SORM</h3>
{% highlight scala %}
Db.save(supplier1)
Db.save(supplier2)
Db.save(supplier3)
{% endhighlight %}
  </li>
  <li>
      <h3>Slick</h3>
{% highlight scala %}
db.withSession {
  Suppliers.insert(101, supplier1.name, supplier1.street, supplier1.city, supplier1.state, supplier1.zip)
  Suppliers.insert(49, supplier2.name, supplier2.street, supplier2.city, supplier2.state, supplier2.zip)
  Suppliers.insert(150, supplier3.name, supplier3.street, supplier3.city, supplier3.state, supplier3.zip)
}
{% endhighlight %}
  </li>
</ul>


##Querying

###Collect Coffee instances having a supplier named "Superior Coffee"
<ul class="vs">
  <li>
      <h3>SORM</h3>
{% highlight scala %}
Db.query[Coffee]
  .whereEqual("supplier.name", "Superior Coffee")
  .fetch()
{% endhighlight %}
  </li>
  <li>
      <h3>Slick</h3>
{% highlight scala %}
db.withSession {
  for {
    s <- Suppliers if s.name === "Superior Coffee"
    c <- Coffees if s.id === c.supID 
  } yield Coffee(c.name, Supplier(s.name, s.street, s.city, s.state, s.zip), c.price, c.sales, c.total)
}
{% endhighlight %}
  </li>
</ul>
###Collect just the names of coffee and supplier 
<ul class="vs">
  <li>
      <h3>SORM</h3>
{% highlight scala %}
Db.query[Coffee].whereSmaller("price", 9.0).fetch()
  .map(c => (c.name, c.supplier.name))
{% endhighlight %}
  </li>
  <li>
      <h3>Slick</h3>
{% highlight scala %}
db.withSession {
  for {
    c <- Coffees if c.price < 9.0
    s <- Suppliers if s.id === c.supID
  } yield (c.name, s.name)
}
{% endhighlight %}
  </li>
</ul>

##Updating
###Update sales and total of an already fetched Coffee
<ul class="vs">
  <li>
      <h3>SORM*</h3>
{% highlight scala %}
Db.save(coffee.copy(sales = coffee.sales + 1, 
                    total = coffee.total + 23.4))
{% endhighlight %}
  </li>
  <li>
      <h3>Slick**</h3>
{% highlight scala %}
db.withSession {
  val q = for(c <- Coffees if c.id === id) 
          yield (c.sales, c.total)
  q.foreach { case (sales, total) =>
    (Coffees.sales ~ Coffees.total)
      .update (sales + 1, total + 23.4)
  }
}
{% endhighlight %}
  </li>
</ul>
<div class="footnotes">
  <p>* SORM automatically identifies the fetched values with help of <a href="/api/#sorm.Persisted"><code>Persisted</code></a> trait. To learn more about how it works please visit <a href="/Documentation.html#persisted_trait_and_ids">this page</a>.</p>
  <p>** In Slick the identifier of the row has to be manually managed, by either extending the model case class to include it, or storing it separately.</p>
</div>

<!-- 
###Update a coffee supplier to be the one having a name "The High Ground"
<ul class="vs">
  <li>
      <h3>SORM</h3>
{% highlight scala %}
val supplier = Db.query[Supplier]
                 .whereEqual("name", "The High Ground")
                 .fetchOne()
supplier.foreach(v => Db.save(coffee.copy(supplier = v)))
{% endhighlight %}
  </li>
  <li>
      <h3>Slick</h3>
      <p>???</p>
  </li>
  </ul>

 -->


<h2>Supported databases</h2>
<ul class="vs">
  <li>
      <h3>SORM</h3>
      <ul>
        <li>MySQL</li>
        <li>PostgreSQL</li>
        <li>H2</li>
        <li>HSQLDB</li>
      </ul>
  </li>
  <li>
      <h3>Slick</h3>
      <ul>
        <li>Derby</li>
        <li>H2</li>
        <li>HSQLDB</li>
        <li>Microsoft Access</li>
        <li>Microsoft SQL Server</li>
        <li>MySQL</li>
        <li>PostgreSQL</li>
        <li>SQLite</li>
      </ul>
  </li>
</ul>
