---
layout: 'content'
title: 'Adding Semantic to Base Types Parameters in Scala'
---

Here is another "fun" dive in the complex type system of Scala. This was inspired by [Eric Torreborre's post](http://etorreborre.blogspot.de/2011/11/practical-uses-for-unboxed-tagged-types.html) that gives a very good use case for what I am going to show, but I wanted to share another one.

I am giving a simple example here, it's a bit naive and there would be other ways to implement it, but I hope you will still understand the overarching concept: being able to annotate the meaning of basic type parameters (and variables) such as `Int`, `List[String]`, etc.
This will help the compiler can automatically detect more errors and a code reviewer to understand better what's going on.


#A Use Case

Imagine that you are trying to get users to score products. Each product can be scored along different dimensions: design, usability, durability.

You could represent a score with a simple object:

```scala
case class User(id: Long)
case class Product(id: Long)

case class Scoring(user: User, product: Product, design: Int, usability: Int, durability: Int)
```

Now, you can create `Scoring` instances in your code `Scoring(u1, p1, 10, 15, 20)`.

You will probably end up creating such objects in different parts of your code, from raw `Int`s coming from diverse sources, such as a REST request, or from parsing a line from your database.

The issue here is that the `Scoring` constructor is simple, but not very safe. If you are like me, you will have to go check in the documentation to know in what order to set the parameters. 

For instance, you might write something like this, to create an instance with Anorm from a database query:

```scala
def parseProduct(row: Row): Product = …

def parseUser(row: Row): User = …

def parseScoring(row: Row): Scoring = Scoring(
        parseProduct(row), 
        parseUser(row),
        row[Int]("design"),
        row[Int]("durability"),
        row[Int]("usability")
)
```

There are two errors in this code:

1. the scala compiler performing type check would straightaway flag the inverted product/user parameters as a compilation error: `type mismatch;  found   : Product required: User`,
2. however, the second error, inverting the `durability` score and the `usability` score will not be spotted by the compiler and even a code reviewer would have to be very concentrated to spot such a trivial error.

#Wrappers

The simple solution would be to create wrapper classes for each type of score:

```scala
case class Design(val score: Int) extends AnyVal
case class Usability(val score: Int) extends AnyVal
case class Durability(val score: Int) extends AnyVal

case class Scoring(user: User, product: Product, design: Design, usability: Usability, durability: Durability)

def parseScoring(row: Row): Scoring = Scoring(
        parseUser(row),
        parseProduct(row), 
        Design(row[Int]("design")),
        Usability(row[Int]("usability")),
        Durability(row[Int]("durability"))
)
```

This makes a bit more code to write, but it explicitly marks the semantic of the `Int`s that you are passing in; making the code easier to understand and the scala compiler will now complain when you create `Scoring` with the scores in the wrong order. If you instantiate a `Usability` score with the `"durability"` column, the compiler won't know the different, but an attentive reviewer would quickly spot the confusion.

Note the `AnyVal` extension on the case classes, this is a new feature of scala 2.10, called [value classes](http://docs.scala-lang.org/overviews/core/value-classes.html). This tells the compiler that these types are just wrappers for typechecking, but that they can be optimized at runtime to their inner value, without the expensive creation of the wrapper object.

This is already quite a good solution and it will go a long way. But imagine that you now want to compute the average of all the scores of a product, you might have a list of `Scoring` for a product and will `foldLeft` on it to sum up every scores. However, because you have included new types, you can't just sum `prod1.design + prod2.design` as the operation `+` is not defined on the `Design` class. You end up having to either write: `Design(prod1.design.score + prod2.design.score)` which makes your code more cumbersome than it should be.

You could also implement a proxy `+` operator in all new wrapper case class and replicate all other operations you might need within the wrapper.
When it's just one operation, for three case classes, that's fine. But if you deal with more complex basic types, such as `String` or composed types, such as `List[Int]` or `Future[Int]`, you might end up having to proxy a lot of operations in your wrappers. If you have loads of such wrappers, it might be a lot of extra code.

#Unboxed Tagged Types

The idea that I have decided to follow, inspired by [Eric's article](http://etorreborre.blogspot.de/2011/11/practical-uses-for-unboxed-tagged-types.html), is to use __"Unboxed Tagged Types"__.
The idea is that you can "tag" types (and not just basic types) with a keyword, without loosing all its operations as you would with a simple wrapper. 

The principle is simple and it is basic object oriented inheritance:  to retain all the features of `Int` you can define a class:

```scala
class DesignScore extends Int
``` 

However, we use a scala trick to make this definition easier and to avoid actually creating new objects: _type aliases_.
 It takes a very simple code scaffold:

```scala
 //code from Eric's article
type Tagged[U] = { type Tag = U }
type @@[T, U] = T with Tagged[U]
```

- `Tagged[U]` defines a new type, which just adds a type property containing the parameter type `U`. I.E it's a type that has an annotation `U`. So if you wrote `type TaggedInt = Int with Tagged[Design]`, you would just define a new type alias,  which extends `Int` and tags it with the new inner type property `type Tag = Design`. 
- `@@[T, U]` is only a convenient shortcut to extend types with the Tagged interface. So you could write the previous example with: `type TaggedInt = Int @@ Design`

You can add this snippet to your own code or use the implementation provided by [scalaz](http://typelevel.org/), which is pretty much equivalent.

Right, this is still pretty abstract; so don't worry if you are not getting it yet. Just see it as a tool to annotate types. Let's see how to use it now with our previous example.

First, we define "tags" that will allow us to annotate the `Int` basic type with some contextual information:

```scala
trait Design
trait Usability
trait Durability
```

These traits don't do anything except existing as a known type for the compiler, you can see them as constants, or values of an enum, at the type level. We can now use them to define our tagged types:

```scala
type DesignScore = Int @@ Design
type UsabilityScore = Int @@ Usability
type DurabilityScore = Int @@ Durability
```

These type aliases just represent an extension of the type `Int` with an embedded type parameter `Tag`. We can use these as we would use any other type, so as for our previous example:

```scala
 case class Scoring(user: User, product: Product,
                    design: DesignScore,
                    usability: UsabilityScore,
                    durability: DurabilityScore)
```

Because `DesignScore` is just a subclass of `Int`, it inherits all its operations, and we can now use the sum operator: `prod1.design+prod2.design`.

However, we are still missing something: scala doesn't know how to create a `DesignScore`, as when you write `val score = 1`, you get an `Int` and scala doesn't know apriori how to convert this to a subtype of `Int`. This actually plays in our favor, as it will force us to explicitely write the conversion when we need to assign an integer to something that needs a `DesignScore`.

Let's write an extension of `Int` that knows how to do this conversion:

```scala
implicit class TaggedInt(val i: Int) extends AnyVal {
     def design = i.asInstanceOf[DesignScore]
     def usability = i.asInstanceOf[UsabilityScore]
     def durability = i.asInstanceOf[DurabilityScore]
}
```

Note a few things:

1. we use a _cast_ to transform a superclass (`Int`) in a subclass (`DesignScore`), it might seem dirty, but it's safe as we will never encounter a case where we cannot perform the case,
2. we are using a new feature of scala 2.10: an _implicit class_ that extends `Int` with the new methods,
3. we shouldn't use a direct _implicit conversion_ between `Int` and `DesignScore` as we would loose the fact that now developers have to be explicit about how they want to use the `Int` type. Nothing would then stop you from using the Usability Int in a DesignScore slot.
4. while I like the postfix operator style, you could just define unary operators if you prefer, without using an implicit class at all: `def design(i: Int): DesignScore = i.asInstanceOf[DesignScore]`.

We can now create the `Scoring` object in the following manner `Scoring(u1, p1, 10.design, 15.usability, 20.durability)`. And in our previous database parsing example:

```scala
def parseScoring(row: Row): Scoring = Scoring(
        parseUser(row),
        parseProduct(row), 
        row[Int]("design").design,
        row[Int]("usability").durability,
        row[Int]("durability").usability
)
```

As you can see, it's already a bit easier to spot the error, and if you don't the compiler will complain anyway.

I hope that through this simplistic example, you have been able to understand how __Unboxed Tagged Types__ can help. Explicitly annotating the use of non semantic basic types (`Int`, `String`, `List[Int]`, …) in your code will have two benefits:

1. in many cases, the compiler will warm you when you are using a value in the wrong slot.
2. by annotating method signatures (_in_ and _out_ types), and the uses of raw types in your code, this one will be easier to understand and you will hopefully make less errors and quickly spot bugs.

#Parameter Validation

As [George Leontiev](http://www.folone.info/blog/Typelevel-Hackery/) points out, you can also use this feature to perform validation.   

In our example, the scores are unbounded, you could pass in any value without anyone complaining. Ideally, you would probably want to have them in a bounded range, let's say between 0 and 5. 

The usual approach would be to validate the input when the user inputs a value. But then, in the rest of the code, there is nothing stopping you from making bad operations on the values.

Clearly, this is not something you can deal with at compile time as you do not know what user inputs you will get or what the values in the database
will be. However, you can build in some gards to warn you when something is wrong.

As we have seen, we cannot build the tagged types without going through the explicit call to the `design`, `durability` or `usability` functions. So we can just extend these functions to add checks on the ranges:

```scala
implicit class TaggedInt(val i: Int) extends AnyVal {
     def design = {
         require(i >= 0 && i <= 5, "the design score has to be between 0 and 5")
         i.asInstanceOf[DesignScore]
     }
     ...
}
```

Now, the validation of your parameters is built-in the "type" that defines it, if you call `Scoring(u1, p1, 10.design, 15.usability, 20.durability)`, an `IllegalArgumentException` will be thrown. If you check for such exception when receiving a score value, you can deal with it properly, report an error to the user, etc.

If you prefer to use the `validation` pattern instead of catching exceptions, you can use _scalaz_ `Validation` monad, or `scala` default `Option` or `Either` constructs:

```scala
implicit class TaggedInt(val i: Int) extends AnyVal {
     def design: Either[String, DesignScore] = {
         if(i >= 0 && i <= 5){
            Right(i.asInstanceOf[DesignScore])
         } else {
            Left("the design score has to be between 0 and 5")
         }
     }
     ...
}

```
