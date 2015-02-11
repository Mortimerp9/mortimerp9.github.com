---
layout: 'content'
title: 'Testing an Option value without the gymnastics'
---


In [Boolean Operators for Option](https://coderwall.com/p/6lwybw), I have shown how to simply build test for the existence of an `Option`. Go check it out, the idea is simple and allows to write:

```scala
val opt1 = Some("string")
val opt2 = None

if(opt1 && opt2) ...
```

instead of the longer `if(opt1.isDefined && opt2.isDefined) ...`. It's just a way to write things in a more straightforward maner.

But what if you want to also test what's in the `Option`? You can use a `match` construct if you have only one option, but when you are mixing different things (like optional parameters), writing a match is a bit cumbersome and `if` is often more direct.

Let's say that you have an optional parameter to a request, which is a string representing a special view. In addition to this optional parameter, you also have to test if you have a valid user ID in the request. You can do something like:

```scala
def request(userID: Long, view: Option[String]) {
    validateUser(userID) match {
        case None => ...
        case Some(user) if view && view.get.equals("details") => ...
        case Some(user) => ...
    }
}
``` 

this `view && view.get.equals("details")` is a bit cumbersome, but `Option` following the collection API, you can use a very nice shortcut to write all this:

```scala
case Some(user) if view.exist(_.equals("details")) => ...
```

The `exist` function only returns true if the Option is `Some` __and__ the predicate you are providing returns true. Here is what the doc says:

> Returns true if this option is nonempty and the predicate p returns true when applied to this scala.Option's value. Otherwise, returns false.

That's it. The `exists` name might be a bit confusing, so you could write an alias:

```scala
  implicit class RichOpt[T](opt: Option[T]) {
    def assert(p: T => Boolean): Boolean = opt.exists(p)
  }
```

or even, if you only deal with `String` in the `Option`:

```scala
implicit class StrOpt(opt: Option[String]) {
  	def strEquals(s: String): Boolean = opt.exists(_.equals(s))
  }
```

that you can use:
```scala
  val opt1 = Some("string")      //> opt1  : Some[String] = Some(string)
  opt1.assert(_.equals("test"))//> res0: Boolean = false
  opt1.strEquals("string")       //> res1: Boolean = true
```
