---
title: SORM vs Slick
group: navigation
layout: page
---
<style>
  table.vs { width: 100%; table-layout: fixed; }
  .vs td { vertical-align: top; width: 100%; }

  .vs td.separator { width: 30px; position: relative; }
  .separator div { position: absolute; top: 36px; bottom: 36px; left: 50%; width: 1px; background-color: #777; }


  .footnotes { margin: 7px 0 14px 14px; }
  .footnotes p { font-size: smaller; line-height: 1.5; padding: 0;  }

  .yes { color: #A6E22E; }
  .no { color: #F92672; }

</style>

In the comparisons presented on this page the following data model will be assumed:

{% highlight scala %}
case class Coffee ( name : String, supplier : Supplier, price : Double, sales : Int, total : Int )
case class Supplier ( name : String, street : String, city : String, state : String, zip : String )
{% endhighlight %}


##Framework initialization
<table class="vs">
  <tr>
    <td>
      <h3>SORM</h3>
{% highlight scala %}
import sorm._

val db = new Instance(
  entities = Set( Entity[Coffee](), Entity[Supplier]() ),
  url = "jdbc:h2:mem:test",
  initMode = InitMode.Create
)
{% endhighlight %}
    </td>
    <td class="separator"><div></div></td>
    <td>
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
  def name = column[String]("COF_NAME", O.PrimaryKey)
  def supID = column[Int]("SUP_ID")
  def price = column[Double]("PRICE")
  def sales = column[Int]("SALES")
  def total = column[Int]("TOTAL")
  def * = name ~ supID ~ price ~ sales ~ total
  def supplier = foreignKey("SUP_FK", supID, Suppliers)(_.id)
}

val db = Database.forURL("jdbc:h2:mem:test", driver = "org.h2.Driver")
{% endhighlight %}
    </td>
  </tr>
</table>

<!-- 
##Connections management and multithreading
<table class="vs">
  <tr>
    <td>
      <h3>SORM</h3>
      <p>Each call to <code>db.connection()</code> opens a new connection to the db. This can be beneficial in multithreaded applications (see <a href="/pages/Connections-and-Multithreading.html" target="_blank">Connections and Multithreading</a>). In other cases opening a single connection as follows will be sufficient.</p>
{% highlight scala %}
val cx = db.connection()
{% endhighlight %}

    </td>
    <td class="separator"><div></div></td>
    <td>
      <h3>Slick</h3>
      <p>By default Slick operates on a single-connection. Using 3rd-party connection poolers multiple connections can be achieved.</p>
      <p>It'll be implied in all subsequent examples that the code is wrapped in the following closure</p>
{% highlight scala %}
db.withSession {
  ...
}
{% endhighlight %}
    </td>
  </tr>
</table>
 -->

##Database schema generation
<table class="vs">
  <tr>
    <td>
      <h3>SORM</h3>
      <p>Automatic</p>
    </td>
    <td class="separator"><div></div></td>
    <td>
      <h3>Slick</h3>
{% highlight scala %}
db.withSession {
  (Suppliers.ddl ++ Coffees.ddl).create
}
{% endhighlight %}
    </td>
  </tr>
</table>


##Inserting

<table class="vs">
  <tr>
    <td>
      <h3>SORM</h3>
{% highlight scala %}
db.withTmpConnection { cx =>
  cx.save(Supplier("Acme, Inc.", "99 Market Street", "Groundsville", "CA", "95199"))
  cx.save(Supplier("Superior Coffee", "1 Party Place", "Mendocino", "CA", "95460"))
  cx.save(Supplier("The High Ground", "100 Coffee Lane", "Meadows", "CA", "93966"))
}
{% endhighlight %}
    </td>
    <td class="separator"><div></div></td>
    <td>
      <h3>Slick</h3>
{% highlight scala %}
db.withSession {
  Suppliers.insert(101, "Acme, Inc.", "99 Market Street", "Groundsville", "CA", "95199")
  Suppliers.insert(49, "Superior Coffee", "1 Party Place", "Mendocino", "CA", "95460")
  Suppliers.insert(150, "The High Ground", "100 Coffee Lane", "Meadows", "CA", "93966")
}
{% endhighlight %}
    </td>
  </tr>
</table>



##Querying

###Collect Coffee instances having a supplier named "Superior Coffee"
<table class="vs">
  <tr>
    <td>
      <h3>SORM</h3>
{% highlight scala %}
db.withTmpConnection { cx =>
  cx.access[Coffee]
    .whereEqual("supplier.name", "Superior Coffee")
    .fetch()
}
{% endhighlight %}
    </td>
    <td class="separator"><div></div></td>
    <td>
      <h3>Slick</h3>
{% highlight scala %}
db.withSession {
  for {
    s <- Suppliers if s.name === "Superior Coffee"
    c <- Coffees if s.id === c.supID 
  } yield Coffee(c.name, Supplier(s.name, s.street, s.city, s.state, s.zip), c.price, c.sales, c.total)
}
{% endhighlight %}
    </td>
  </tr>
</table>

###Collect just names of coffee and supplier 
<table class="vs">
  <tr>
    <td>
      <h3>SORM</h3>
{% highlight scala %}
db.withTmpConnection { cx =>
  cx.access[Coffee].whereSmaller("price", 9.0).fetch()
    .map(coffee => coffee.name -> coffee.supplier.name)
}
{% endhighlight %}
    </td>
    <td class="separator"><div></div></td>
    <td>
      <h3>Slick</h3>
{% highlight scala %}
db.withSession {
  for {
    c <- Coffees if c.price < 9.0
    s <- Suppliers if s.id === c.supID
  } yield (c.name, s.name)
}
{% endhighlight %}
    </td>
  </tr>
</table>


##Updating
###Update sales and total of an already fetched Coffee
<table class="vs">
  <tr>
    <td>
      <h3>SORM*</h3>
{% highlight scala %}
db.withTmpConnection { cx =>
  cx.save(coffee.copy(sales = coffee.sales + 1, 
                      total = coffee.total + 23.4))
}
{% endhighlight %}
    </td>
    <td class="separator"><div></div></td>
    <td>
      <h3>Slick**</h3>
{% highlight scala %}
db.withSession {
  val q = for(c <- Coffees if c.id === id) 
          yield (c.sales, c.total)
  q.foreach { case (sales, total) =>
    Coffees.sales ~ Coffees.total update (sales + 1, 
                                          total + 23.4)
  }
}
{% endhighlight %}
    </td>
  </tr>
</table>
<div class="footnotes">
  <p>* SORM automatically identifies the fetched values with help of `Persisted` trait.</p>
  <p>** In Slick the identifier of the row has to be manually managed, by either extending the model case class to include it, or storing it separately.</p>
</div>


###Update a coffee supplier to be the one having a name "The High Ground"
<table class="vs">
  <tr>
    <td>
      <h3>SORM</h3>
{% highlight scala %}
db.withTmpConnection { cx =>
  val supplier = cx.access[Supplier]
                   .whereEqual("name", "The High Ground")
                   .fetchOne()
  supplier.foreach(s => cx.save(coffee.copy(supplier = s)))
}
{% endhighlight %}
    </td>
    <td class="separator"><div></div></td>
    <td>
      <h3>Slick</h3>
      <p>???</p>
    </td>
  </tr>
</table>





<h2>Supported databases</h2>
<table class="vs">
  <tr>
    <td>
      <h3>SORM*</h3>
      <ul>
        <li>H2</li>
        <li>MySQL</li>
      </ul>
    </td>
    <td class="separator"><div></div></td>
    <td>
      <h3>Slick</h3>
      <ul>
        <li>Derby/JavaDB</li>
        <li>H2</li>
        <li>HSQLDB/HyperSQL</li>
        <li>Microsoft Access</li>
        <li>Microsoft SQL Server</li>
        <li>MySQL</li>
        <li>PostgreSQL</li>
        <li>SQLite</li>
      </ul>
    </td>
  </tr>
</table>
<div class="footnotes">
  <p>* Support for more dbs is comming in nearest releases of SORM</p>
</div>



<h2>Feature-wise comparison</h2>
<table border="1" cellpadding="3">
  <tr>
    <td width="300"><h3>Supported Feature</h3></td>
    <td><h3>SORM</h3></td>
    <td><h3>Slick</h3></td>
  </tr>
  <tr>
    <td><h4>Abstraction from Relations (one-to-many, one-to-one)</h4></td>
    <td class="yes">Yes</td>
    <td class="no">No</td>
  </tr>
  <tr>
    <td><h4>Abstraction from Insert and Update Operations</h4></td>
    <td class="yes">Yes</td>
    <td class="no">No</td>
  </tr>
  <tr>
    <td><h4>Scala Collections (Seq, Set, Map)</h4></td>
    <td class="yes">Yes</td>
    <td class="no">No</td>
  </tr>
  <tr>
    <td><h4>Scala Option</h4></td>
    <td class="yes">Yes</td>
    <td>???</td>
  </tr>
  <tr>
    <td><h4>Scala Tuple</h4></td>
    <td class="yes">Yes</td>
    <td class="no">No</td>
  </tr>
  <tr>
    <td><h4>Scala Enumeration</h4></td>
    <td class="yes">Yes</td>
    <td class="no">No</td>
  </tr>
  <tr>
    <td><h4>Joda Time Dates</h4></td>
    <td class="yes">Yes</td>
    <td class="no">No</td>
  </tr>
  <tr>
    <td><h4>Java Sql Dates</h4></td>
    <td class="no">No</td>
    <td class="yes">Yes</td>
  </tr>
  <tr>
    <td><h4>Transactions</h4></td>
    <td class="yes">Yes</td>
    <td class="yes">Yes</td>
  </tr>
  <tr>
    <td><h4>Raw SQL Queries</h4></td>
    <td class="no">No*</td>
    <td class="yes">Yes</td>
  </tr>
  <tr>
    <td><h4>Native DB Functions</h4></td>
    <td class="no">No</td>
    <td class="yes">Yes</td>
  </tr>
  <tr>
    <td><h4>Native DB Functions</h4></td>
    <td class="no">No</td>
    <td class="yes">Yes</td>
  </tr>
  <tr>
    <td><h4>Statically Resolved Query Parameters</h4></td>
    <td class="no">No</td>
    <td class="yes">Yes</td>
  </tr>
</table>
<div class="footnotes">
  <p>* Provides an effective <a href="{{BASE_PATH}}/api/#sorm.Connection">fetchById</a> method, which accepts an id parameter, which you can manually fetch using, for example, the standard JDBC API.</p>
</div>
