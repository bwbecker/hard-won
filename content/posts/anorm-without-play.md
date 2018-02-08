---
title: "Using Anorm without Play"
date: 2015-05-05T09:05:05-05:00
draft: false
categories: 
  - "anorm" 
  - "postgres"
  - "scala"
keywords: Play!, anorm, postgres
description: "Using Anorm without the Play! framework."

---

Ever want to use Anorm to access a database without all the overhead of Play?  
I did in my 
[previous post](../postgres-enums-anorm/) 
where I played with Postgres enumerations.  Here's the code I used.

It has a couple of features:

*	It reads the `.pg_service.conf` and `.pgpass` files from your home directory to find
a service definition and the appropriate passwords to use for the connection.  This
keeps passwords and such out of your code and out of your repository.
*	It mimics the `DB` class in Anorm to provide a database connection that you can
then use with Anorm.

## DB class

The `DB` class provides three methods:  two that get a connection and one that mimics `withConnection`
([scaladoc](https://www.playframework.com/documentation/2.3.x/api/scala/index.html#play.api.db.DB$)).
`withConnection` is the one I use the most (by far) because it handles closing the connection.

`DB` assumes the postgres driver is available in the class path and uses `PgService` (see below) to
get the connection data.

Typical usage is as follows:

``` scala
val db = DB("local_dev")

db.withConnection { implicit conn =>

    val sql = SQL"""insert into _oat.test_enum (note_category, sort_order) 
            VALUES  (${NoteCategory.Advisor}, ${SortOrder.Ascending}) 
            RETURNING id"""
    val id = sql.as(anorm.SqlParser.scalar[Long].singleOpt) 
}
```

The "local_dev" that is passed to the `DB` constructor is the name of the postgres service
to find in the .pg_service.conf file.

And, here's the code for `DB`:

``` scala
package oatLib.db

import java.sql.DriverManager
import java.sql.Connection

/**
 * Lots of this is stolen from Play.
 */
case class DB(service:String) {

  private val pgService = PgService(service)
  
  private val url = s"jdbc:postgresql://${pgService.host}:${pgService.port}/${pgService.dbname}"
  private val user = pgService.user
  private val password = pgService.password

  Class.forName("org.postgresql.Driver").newInstance


  /**
   * Retrieves a JDBC connection.
   *
   * Don't forget to release the connection at some point by calling close().
   *
   * @return a JDBC connection
   * @throws an error if the required data source is not registered
   */
  def getConnection(): Connection = {
    var props = new java.util.Properties();
    props.setProperty("user", user);
    props.setProperty("password", password);

    DriverManager.getConnection(url, props)
  }

  /**
   * Retrieves a JDBC connection.
   *
   * Don't forget to release the connection at some point by calling close().
   *
   * @param autocommit when `true`, sets this connection to auto-commit
   * @return a JDBC connection
   * @throws an error if the required data source is not registered
   */
  def getConnection(autocommit: Boolean = true): Connection = {
    val connection = this.getConnection
    connection.setAutoCommit(autocommit)
    connection
  }

  /**
   * Execute a block of code, providing a JDBC connection. The connection and all created statements are
   * automatically released.
   *
   * @param name The datasource name.
   * @param block Code block to execute.
   */
  def withConnection[A](block: Connection ⇒ A): A = {
    val connection = getConnection
    try {
      block(connection)
    } finally {
      connection.close()
    }
  }
}
```

## PgService class

The `apply` method looks in your home directory for the `.pg_service.conf` and 
`.pgpass` files.  The postgres programs that use `.pgpass` have a sophisticated
matching algorithm to choose the specific password required based on the 
database, user, port, etc.  I doubt that I've completely reverse engineered
that algorithm, but I believe this comes pretty close.

``` scala
package oatLib.db

import scala.io.{ Source }

/**
 * Read the service information from the account's pg_service.conf
 * and pgpass files.
 *
 * It assumes they're are ~/.pg_service.conf and ~/.pgpass.
 *
 */
case class PgService(service: String,
                     host: String,
                     port: Int,
                     dbname: String,
                     user: String,
                     password: String)

object PgService {

  /**
   *  Get the details for the named service from the combination of
   *  the service source (svcFile) and the password source (pwdFile).
   */
  def apply(service: String, svcFile: Source, pwdFile: Source): PgService = {

    def getService: Map[String, String] = {
      // Suck in the services file, get rid of services before the one
      // we want, take the one we want, turn it into a map of key-value
      // pairs.
      val allSvc = svcFile.getLines.toList
      val dropLeadingSvc = allSvc.dropWhile(line ⇒ line != s"[$service]")
        .dropWhile(line ⇒ line == s"[$service]")
      val svcDef = dropLeadingSvc.takeWhile(line ⇒ line.matches("[^=]+=[^=]+"))
      val svcDef2 = svcDef.map(line ⇒ line.split('=')).map(a ⇒ (a(0), a(1)))
      svcDef2.toMap
    }

    val props = getService
    if (props.isEmpty) {
      throw new Exception(s"Unable to find a service configuration for $service")      
    }
    val host = props("host")
    val port = props("port").toInt
    val dbname = props("dbname")
    val user = props("user")

    val pwCandidates = pwdFile.getLines.toList
      .filter(_.matches(s"^([^:]*:){0}($host|\\*):.*")) // matches host
      .filter(_.matches(s"^([^:]*:){1}($port|\\*):.*")) // matches port
      .filter(_.matches(s"^([^:]*:){2}($dbname|\\*):.*")) // matches dbname
      .filter(_.matches(s"^([^:]*:){3}($user|\\*):.*")) // matches user

    if (pwCandidates.isEmpty) {
      throw new Exception(s"Unable to find a password for $host:$port:$dbname:$user")
    }
    val password = pwCandidates.headOption.map(_.split(':')(4)).getOrElse("")
    new PgService(service, host, port, dbname, user, password)
  }

  /**
   * Get the details for the named service from the default config
   * files (~/.pg_service.conf and ~/.pgpass).
   */
  def apply(service: String): PgService =
    apply(service,
      Source.fromFile(sys.env("HOME") + "/.pg_service.conf"),
      Source.fromFile(sys.env("HOME") + "/.pgpass"))
}
```

That's it!  Enjoy!
