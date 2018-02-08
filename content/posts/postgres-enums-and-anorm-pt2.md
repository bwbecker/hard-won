---
title: "Postgres Enums and Anorm, Part 2"
date: 2015-05-06T08:59:49-05:00
draft: false
---

I went to implement my enumerations discoveries as chronicalled in 
[Postgres Enums and Anorm](../postgres-enums-anorm/)
and realized anew something that had niggled away in the back of my brain:  I'm working
with data from a legacy database and most of the enumerations are indecipherable.  Many of 
the enumerations are like this:

``` sql
CREATE TYPE _quest.quest_instruction_mode AS ENUM
   ('P',
    'CO');
```

where 'P' stands for (as far as we can tell!) "in-Person" and 'CO' stands for "Course-Online".

So naturally, I'd like an intelligible Scala enumeration such as 

``` scala 
object InstructionMode extends DbEnum {
  type InstructionMode = Value

  val InPerson = Value("P")
  val Online = Value("CO")
}
```

But here's the rub:  `println(InstructionMode.Online)` still prints the indecipherable `CO`.


The goals I'm pursuing are:

* Easy interoperability between Posgres enumerations and Scala enumerations.
* Printing meaningful values from my program rather than the obscure codes kept in the database.
* Minimal code duplication or bloat.

Here's my solution.  As in my [previous post](http://bwbecker.github.io/blog/2015/05/05/postgres-enums-and-anorm/),
I'm adding a common superclass to my enumerations that itself extends Scala's `Enumeration` class.  The difference
this time is that I've defined an abstract type, `myType`, and an abstract list that will contain the codes
actually used in the database.  This allows me to do almost all of the work in the superclass, provided I
have one function: `unapply` takes a string (the one stored in the database) and returns a value from the
Scala enumeration.

A typical Scala enumeration now looks like this:

``` scala
object InstructionMode extends DbEnum {
  type InstructionMode = Value

  val InPerson, Online = Value

  protected type myType = Value
  protected val dbValues = Array("P", "CO")
  protected def unapply(s: String): myType = InstructionMode(dbValues.indexOf(s))
}
```

Notes:

*	Line 1 extends `DbEnum` rather than `Enumeration`.
*	Thanks to line 4, printing the enumeration's values will give a meaningful result, 
either "InPerson" or "Online".
*	Line 6 defines the abstract type we'll need in `DbEnum`.  This line is the same in every enum but 
is needed because `Value` is different for every enum.
*	Line 7 defines the values actually contained in the database.  They need to be in 
the same order as the values specified in line 4.
*	Line 8 is the `unapply` function.  In all of my cases, it's exactly as shown here
except for the obvious substitution for `InstructionMode`.

The requirement that the order of values in lines 4 and 7 match is a problem, in my mind.
There's no question that this can be a source of bugs.  But I'm out of ideas for how to
improve it.  If you have some, please comment!  I'd also appreciate insight into how
the name `InPerson` is captured and put into a map in the `Enumeration` class.  There's
still something going on there that I don't understand but I'd like to!

Finally, the code for `DbEnum` is:

``` scala 
abstract class DbEnum extends Enumeration {

  protected type myType <: Enumeration#Value
  protected val dbValues: Array[String]
  protected def unapply(s: String): myType

  /**
   * Create an implicit to help with converting this Scala enum into the equivalent
   * Postgres enum.
   */
  implicit val toStatement = new ToStatement[myType] {
    def set(s: java.sql.PreparedStatement, index: Int, aValue: myType): Unit = {
      s.setObject(index, dbValues(aValue.id), java.sql.Types.OTHER)
    }
  }

  /**
   * Convert a database enumeration to a Scala enumeration.
   * @param convert A conversion function from a string (the value received from the database) to E (the Scala enum).
   */
  implicit def enumToType(implicit m: Manifest[myType]): Column[myType] = Column {
    (value, meta) =>
      val MetaDataItem(qualified, nullable, clazz) = meta

      try {
        val s = value.asInstanceOf[String]
        eitherToError(Right(unapply(s))): MayErr[SqlRequestError, myType]
      } catch {
        case e: Exception =>
          eitherToError(Left(TypeDoesNotMatch("Cannot convert " + value + ":" +
            value.asInstanceOf[AnyRef].getClass + " to " +
            m.runtimeClass.getSimpleName + " for column " +
            qualified)))
      }
  }

  /**
   * Create a json Reads to read instances of this enumeration from JSON.
   */
  implicit val reads = new Reads[myType] {
    def reads(json: JsValue): JsResult[myType] = json match {
      case s: JsString =>
        val enum = unapply(s.value)
        JsSuccess(enum)
      case x => JsError(s"Expected a string; got $x.")
    }
  }
}
```

*	Lines 3-5 define the abstract members that need to be defined in each 
of the individual enumerations.
*	The definitions of `toStatement` and `enumToType` are very much the same
as in the [previous post](http://bwbecker.github.io/blog/2015/05/05/postgres-enums-and-anorm/)
except that with the new definition of `myType` we have all the information to move everything
into `DbEnum`.
*	This implementation contains a bonus:  Lines 40-47 contain a function that reads
the enumeration from a json blob.  That's also central to how I'm dealing with my database
and will, no doubt, be the subject of a future post.

That's it!  Enjoy!
