---
title: Connections and Multithreading
group: doc
layout: page
---

Most desktop applications don't require


, i.e., for instance, a single connection to a MySQL db

For instance, a MySQL server always runs a connection on a single thread and therefore with a maximum load on it MySQL utilizes a power of only a single processor at max. So, to utilize the MySQL multithreading capabilities your application needs to keep multiple connections to the db-server. This is the way to do scaling in that respect. SORM 0.2 was designed with an Actor-based multithreading model in mind. 

If your application is not database-intensive and you don't need scaling you can simply use just a single connection


But the ones that need scaling require


The [Connection](http://nikita-volkov.github.com/sorm/api/#sorm.Connection) API 




But in certain applications where




SORM was designed with Actor-based multithreading model in mind. 



import sext._, embrace._
import embrace._
import akka.actor.Actor
import radiox.amazon_albums_spider._
import Scraping._
import Domain._
import sorm._


object DbNegotiator {
  case object StartTask
  case class CloseTask ( task : Task )
  case class CreateTasks ( asins : Seq[String] )
  case class SaveAlbum ( album : Album )
  case object InitializeTasks

  case class TaskStarted ( task : Task )
  case object NoTasksToStart
}
class DbNegotiator ( protected val db : Connection ) extends Actor with DbNegotiatorTasks {
  import DbNegotiator._


  override def postStop() { db.close() }

  def receive = {
    case InitializeTasks =>
      if( !db.access[Task].exists() ){
        db.save( Task("B008FSCNTK", db.now()) )
        db.save( Task("B00122X4JE", db.now()) )
      }

    case SaveAlbum(album) =>
      album $ save $ sender.!

    case StartTask =>
      startTask() match {
        case None => sender ! NoTasksToStart
        case Some(task) => sender ! TaskStarted(task)
      }

    case CloseTask(task) =>
      db.save(task.copy(closed = db.now() $ (Some(_))))

    case CreateTasks(asins) =>
      db.transaction {
        val existingTasks = db.access[Task].whereIn("asin", asins).fetch().toStream
        // reset the failed generated ones
        existingTasks.filter(_.failure != None).filter(_.generated).map(_.copy(failure = None, generated = false, outdated = false)).foreach(db.save)
        // create the new ones
        asins.view.diff(existingTasks.map(_.asin)).foreach( Task(_, db.now()) $ db.save )
      }
  }