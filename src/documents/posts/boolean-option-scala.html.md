---
layout: 'content'
title: 'Boolean Operators for Options'
---


Options in scala are very useful and help you explicitly specify if your methods might return no value (where you would return `null` in Java) and when parameters are optional.

For instance, you might have a REST endpoint with something like this:

```scala
def users(nameFilter: Option[String], cityFilter: Option[String])
```

Option are usually managed with `match`, `map` and `flatMap`, or for comprehensions. But imagine you want to do a simple test, to know if you have at least one filter, `match` would be cumbersome, as you will have to define loads of cases. But you can do this with a simple test:

```scala
val withFilter = nameFilter.isDefined || cityFilter.isDefined
```

If you start doing this with many Options, you end up with a lot of `isDefined` and your code becomes a bit less readable than it could be. Wouldn't it be good if you could just do: `nameFilter || cityFilter`, after all Options are very similar to Booleans, they are either "true" (defined), or not.

With scala implicit conversion, this is very simple, we can just convert option to Boolean: 


```scala
implicit def optToBool(opt: Option[_]): Boolean = opt.isDefined
```

This will give:

```scala

Some(10) || None         //> res0: Boolean = true
Some(10) || Some(20)  //> res1: Boolean = true
None || Some(20)         //> res2: Boolean = true
None || None                //> res3: Boolean = false

Some(10) && Some(20)  //> res4: Boolean = true
Some(10) && None         //> res5: Boolean = false
None && Some(20)        //> res6: Boolean = false
None && None               //> res7: Boolean = false

```
