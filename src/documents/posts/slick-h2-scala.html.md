---
layout: 'content'
title: 'Proper configuration of H2 with Scala Slick for testing'
---

Using [H2](www.h2database.com) in memory database for running slick tests is tempting, but if you use a naive configuration, you will end up with very odd exceptions such as `Column "X2.bar" not found;` or `Table "foo" not found; SQL statement:`. 

There are two causes to these exception, and it most probably isn't an error in your code, but just a configuration trick that you missed. I eventually found the solution in a [(non) bug report on slick](https://github.com/slick/slick/issues/166).

The first issue, is that if you create the tables in one session and run your tests in another session, something like: 

```scala
    test("Foo should Bar") {

        val conn = SlickDatabase.forURL("jdbc:h2:mem:test", driver = "org.h2.Driver")

        conn.withSession {
          import scala.slick.session.Database.threadLocalSession
          val t = MyTable
          t.ddl.create     
        }

       Foo.bar(conn) should be("foobar")
    } 
```

where `Foo.bar` uses its own session to run things. The in memory database will only live as long as the first session, so the `Foo.bar` call will see an "new" empty database. To get around this, you need to tell H2 to keep the database in memory as long as the JVM lives and not as long as the session lives:

```scala
    val conn = SlickDatabase.forURL("jdbc:h2:mem:test;DB_CLOSE_DELAY=-1", driver = "org.h2.Driver")
```

This will go around the first `Table not found` exception, but the code won't work straight away if you used lower case names for your table and columns as H2 tends to uppercase everything by default. That's where finding what's going on is not straightforward. The solution is simple though, you need to tell H2 not to uppercase the world:

```scala
val conn = SlickDatabase.forURL("jdbc:h2:mem:test;DATABASE_TO_UPPER=false;DB_CLOSE_DELAY=-1", driver = "org.h2.Driver")
```
