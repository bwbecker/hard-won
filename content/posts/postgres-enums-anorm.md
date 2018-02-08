---
title: "Postgres Enums and Anorm"
date: 2015-05-05T08:59:49-05:00
draft: false
categories: 
    - "postgres"
    - "anorm" 
    - "scala" 
    - "play" 
    - "enumeration"
description: "How to marry enumerations in a Postgres database with a Scala program via the Anorm database access library."
keywords: "Play!, playframework, posgresql, scala, enumeration, anorm"

---

Types are a wonderful thing in programming languages.  It's the main reason 
I'm using Scala.  Having the compiler do a proof that at least part of my 
program is correct is a huge advantage over a dynamically typed language.

Databases can also make use of types -- with similar advantages.  In my current
project we've been 
receiving data from a database that does not use types or other constraints 
that are the hallmark of modern database design. What a mess!  Tables with 
invalid values, or columns that are never null but nulls are nevertheless allowed, 
or a column that allows nulls but represents them as either the empty string or a 
string with one space in it rather than good 'ole `null`.

This blog post considers how to marry enumerations in a Postgres database 
with a Scala program via the Anorm database access library.

 >   Spoiler:  This is not what I actually implemented.  After you read
 >   this, be sure to read the [follow-up](../postgres-enums-and-anorm-pt2/)

## Defining Enumerations

Let's start with defining enumerations.  The following SQL does it in 
Postgres:

**Aside:** You'll note a couple of oddities in the SQL.  They're vestiages of
the project I'm working on.  `_oat` is the database schema I use most frequently.  `std_note_category` and `oat_sort_order` are actual enums from my current project.

``` sql
CREATE TYPE _oat.std_note_category AS ENUM
   ('Auto',
    'Advisor',
    'CourseEntry');

CREATE TYPE _oat.oat_sort_order AS ENUM
   ('ASC',
    'DESC');
```

We can create a quick test table with

``` sql
CREATE TABLE _oat.test_enum (
    id               SERIAL,
    note_category     _oat.std_note_category not null,
    sort_order        _oat.oat_sort_order
);
```

Note that one of columns is nullable, the other is not.

The database enumeration is mirrored by two Scala enumerations.  The first 
one uses names that match the values in the database.  The second one 
uses more descriptive names.

``` scala
object NoteCategory extends Enumeration {
    type NoteCategory = Value

    val Auto, Advisor, CourseEntry = Value
}

object SortOrder extends Enumeration {
    type SortOrder = Value

    val Ascending = Value("ASC")
    val Descending = Value("DESC")
}
```

## First Attempt:  casting in the queries

We can covert back and forth between the Scala enumerations and the 
database enumerations, but it's painful.  Works like this:

``` scala
db.withConnection { implicit conn =>

    // Delete everything from the table
    SQL"""truncate _oat.test_enum""".execute()

    // Insert one of each
    val sql = SQL"""insert into _oat.test_enum 
            (note_category, sort_order) VALUES  
                (${NoteCategory.Advisor.toString}::_oat.std_note_category,
                 ${SortOrder.Ascending.toString}::_oat.oat_sort_order
                ) RETURNING id"""
    val id = sql.as(anorm.SqlParser.scalar[Long].singleOpt) 

    // Read them back and verify
    val r = SQL"""select * 
                    from _oat.test_enum 
                    where id = ${id}""".apply().head
    val (nc, so) = (r[String]("note_category"), 
                    r[Option[String]]("sort_order"))
    assert(NoteCategory.withName(nc) == NoteCategory.Advisor)
    assert(SortOrder.withName(so.get) == SortOrder.Ascending)
}
```

The objections to this code include:

0. The call to `toString` in lines 9 and 10.
0. The explicit casts required in those same lines.
0. Reading the values back again as strings in line 18 and 19.
0. Explicitly converting the strings to enums in lines 20 and 21. 

Obviously, we want to do better.

But before we dive into improving, a couple of things to note:

* `withName` (lines 20, 21) converts a string like "Auto" into the corresponding enumeration value.
* In line 19 we use `Option[String]` because the column is nullable.

## Second Attempt:  Using ToStatement and a Column converter

Anorm uses implicit functions to assist in converting to and from SQL 
statements.  We'll start with the `ToStatement`, which allows us to embed 
Scala enumerations in queries easily.  We need a couple of functions in 
each Scala enumeration we write, so put `createEnumToStatement` in a new superclass
and then change the enumerations to extend that class:

``` scala
class DbEnum extends Enumeration {
    protected def createEnumToStatement[E]() = new ToStatement[E] {
        def set(s: java.sql.PreparedStatement, index: Int, aValue: E): Unit = {
            s.setObject(index, aValue.toString, java.sql.Types.OTHER)   
        }   
    }
}

object NoteCategory extends DbEnum {   // Extend DbEnum instead of Enumeration
    ...		// same as before

    implicit val noteCategoryToStatement = createEnumToStatement[NoteCategory]()
}
```

Line 12 is the important one.  When you `import NoteCategory._` to bring the enum values into 
scope, this implicit is also brought into scope and used in the SQL statement to interpolate
a `NoteCategory` value into the SQL.  The function's name 
doesn't matter.  You'll need a similar line for the other enum, of course.

Line 12 is what allows us to drop the casts and the explicit calls to `toString` in the 
insertion SQL, above.  Replace it with the following:

``` scala
    val sql = SQL"""insert into _oat.test_enum (note_category, sort_order) 
            VALUES  (${NoteCategory.Advisor}, ${SortOrder.Ascending}) RETURNING id"""
```

`createEnumToStatement` creates a new object that sets the appropriate value in a JDBC
prepared statement.  If you look at the JavaDoc for `java.sql.PreparedStatement` you'll see
many `set` statements:  `setInt`, `setBoolean`, `setTime`, etc.  Unfortunately, `setEnum` is
not one of them.

The `setObject` method is interesting.  It includes an extra parameter that allows you to 
specify the type of the value.  As you can see in line 4, we use it to pass the enumeration
as a string but with the type of `java.sql.Types.OTHER`.  Postgres uses that as a clue to
consult the type of the column to do the right thing.

### Adding a Column converter

The last step is to automatically convert a Postgres enum read from the database into
a Scala enum.  This is done with a `Column` converter.  Its use parallels that of `createEnumToStatement`:

``` scala
class DbEnum extends Enumeration {
    protected def enumToType[E](convert: String => E)(implicit m: Manifest[E]): Column[E] = Column {
        (value, meta) =>
            val MetaDataItem(qualified, nullable, clazz) = meta

            try { 
                val s = value.asInstanceOf[String]
                eitherToError(Right(convert(s))): MayErr[SqlRequestError, E]
            } catch {
              case e: Exception => 
                eitherToError(Left(TypeDoesNotMatch("Cannot convert " + value + ":" + 
                    value.asInstanceOf[AnyRef].getClass + " to " + 
                    m.runtimeClass.getSimpleName + "for column " + 
                    qualified)))
           }
    }
    // createEnumToStatement, as before
}

object NoteCategory extends DbEnum {
	// define enum values and use createEnumToStatement, as before

    implicit val rowToNoteCategory = enumToType[NoteCategory](NoteCategory.withName)
}
```

Some explanatory comments:

0. `enumToType` is again parameterized by the type of the enum.
0.	In line 2 it consumes a function that converts a string (the value read from the database)
into a value of type E (the enum).  For most enumerations this is simply the `withName` function
that we get for free when defining the enum.  And that is, indeed, what's passed in line 23.
0.	Things can go wrong in two ways:  we might get an unexpected value from the database that
can't be converted by `withName` (probably indicating that your Postgres enum and Scala enum are
out of synch) and, if things are really borked, we might not even get a string from the database.
Either of these are caught and turned into an error in lines 11-14.
0.	If the column should never have a null value, you can wrap `enumToType` in line 23 with 
`Column.nonNull(enumToType...)`.  This will throw the familiar "Unexpected nullable" error if 
a null value is found.  If nulls are expected, then read the value as an `Option[SortOrder]`, for example.

## Final Comments

The [test program](http://bwbecker.github.io/downloads/code/postgres_enums.scala) is 
self-contained except for the first 
import line.  That's what provides some support code to make the database connection.  I'll 
write that up soon.

The other imports are 

``` scala
import anorm._
import org.postgresql.util.PGobject
import anorm.MayErr._
import java.sql.Connection
```

That's it!  Enjoy!
