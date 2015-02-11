---
layout: 'content'
title: 'Safely parsing Strings to numbers in Scala'
---

#Too much boilerplate

In Java and Scala, you can convert `String` to numbers pretty easily, but as you might imagine, not all strings will convert to a number. In both languages, to safely convert your strings, you need to be ready to catch a `NumberFormatException`. This often creates quite a lot of boilerplate, even in scala.

Once you have caught that exception, it's not clear what to return in the code. In java, you would either return a null, or some default value, but `null` is evil and it's giving a lot of responsibility to a function to let it decide what is the right default value. The solution to this in Scala is to return an `Option` and return `None` in case of an exception (or use `Either` if you want more info about your exception). You end up with a code like this:

```scala
def safeStringToInt(str: String): Option[Int] = try {
   Some(str.toInt)
} catch {
   case NumberFormatException => None
}
```

But seriously, that's a lot of boilerplate for a simple operation that is used pretty often. However, scala creators thought of this kind of cases and provide a nice little tool to do such `try/catch` that returns an `Option`, and it's all in the [Exception util class][1]:

```scala
def safeStringToInt(str: String): Option[Int] = {
    import scala.util.control.Exception._
    catching(classOf[NumberFormatException]) opt str.toInt
}
```

That's already a lot better!

#Make it an util

While it's a lot less boilerplate, you need to have this method in your context somewhere and then call it. If you come form Java, you would think of doing this with a static method in some util class. In Scala, there is a much neatier way of doing it: implicits!

Ok, in my two last posts, I complained about implicits, but I was talking about implicit arguments. There is actually some cool implicits in scala:
__Implicit conversions__ allow you to transparently add postfix function to existing types, without having to modify their sources. In scala 2.10, this has been made a lot easier with _implicit classes_:

```scala
object StringUtils {
     implicit class StringImprovements(val s: String) {
         import scala.util.control.Exception._
         def toIntOpt = catching(classOf[NumberFormatException]) opt s.toInt
     }
}
```

You can then `import StringUtils._` and call toIntOpt on any String:
```scala
"1234".toIntOpt //> Some(1234)
"lkdsfasd".toIntOpt //> None
```


 [1]: http://www.scala-lang.org/api/current/index.html#scala.util.control.Exception$
