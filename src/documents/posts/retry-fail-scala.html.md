---
layout: 'content'
title: 'Retry a fail-able block in Scala'
---

I have seen a few implementations of this online, in particular [Retrying with style](http://www.scalasteps.com/2011/05/retrying-with-style.html), which is a two years old post and has a few things that bothered me: 

1. it's blocking, Futures where not in scala at the time,
2. it uses mutable variables, which are not intrinsically bad in such a contained context; but my editor highlights them in red and I don't like having red bits in my code :D

I have thus created a [new implementation available at this gist](https://gist.github.com/Mortimerp9/5430595).

The idea is that you could have a block of code that fails for any kind of reasons which are non deterministic. For instance, a call to the DB could fail because the connection queue is full, or a request to a webservice could fail because the "internet is busy".

Instead of writting a loop and a lot of code relevant to the retry part, you can write a block wrapper in scala that will take care of everything in the background for you:

```scala
val myResult = retry(10) {
   ...
   makeWSCall()
   ...
}
```

This is what the little bit of code that I have put in [this gist](https://gist.github.com/Mortimerp9/5430595) does. I won't copy all of it here as it's too long, but here are some comments on how it works:

- I use a `Promise` to eventually embed a value in a `Future`. The future of that promise is returned directly and the tries are executed asynchronously.
- if an exception is caught, it tries again, unless it has exhausted the given number of retries, in which case it fails the future.
- as soon as the result can be computed without exceptions, it is returned in the future.

Because the block returns a `Future`, the retries are executed asynchronously, hopefully not blocking your data flow. You can use the `Future` API to manipulate the value eventually returned:

```scala
myResult.map(_ * 2)
```

You can also use the built-in recovery methods of `Future` to deal with a failed result (too many retries):

```scala
val myResult = retry(10) {
   ...
   makeWSCall()
   ...
} recover {
  case t: Throwable => 0
}
```

The `retry` function also provides some advanced optional parameters:

- in addition to the mandatory maximum retry count, you can define an optional deadline after which it should just give up:

```scala
val myResult = retry(10, Some(10 seconds fromNow)) {
   ...
   makeWSCall()
   ...
}
```

- by default, the block will be retried with an [exponential back-off](en.wikipedia.org/wiki/Exponential_backoff), as it is often used to deal with resources that don't like to be overloaded, you can change this default if you want

```scala
val myResult = retry(10, backoff = (r) => 100 milliseconds) {
   ...
   makeWSCall()
   ...
}
```

- if you want the block to fail without retry for particular exceptions, you can specify a filtering function with the `ignoreThrowable` optional parameter


