---
layout: 'content'
title: 'Easier access to Configuration values in with Typesafe Config'
---

It all started when I was trying to configure a set of scheduled tasks for [Play! 2][1], and it got me launched on a "scrap the boilerplate" mission in my application configuration.

With Play, you can use `Akka.system.scheduler.schedule` to launch recurring background tasks within your web application. To do this, you need to provide two `Duration` that define:
1. when to start the scheduling
2. how long to wait before repeating the task

That's something that you wouldn't want hard coded, and Play provides [a nice way to read your `application.conf`][2] file and parse durations. In fact, you can write something like

```
myapp.notifier.scheduleStart = 2 minutes
myapp.notifier.scheduleEvery = 1 hour
```

and use:

```scala
import Play.current
Play.configuration.getMilliseconds("myapp.notifier.scheduleStart")
```

which will return an `Option[Long]` wich maybe contains the configuration. So you also have to provide a default in your code, so really get a duration, you end up writting:

```scala 
import Play.current
import scala.concurrent.duration._
Play.configuration.getMilliseconds("myapp.notifier.scheduleEvery").getOrElse(30000l) milliseconds
```

The `getOrElse` is there to provide the default value in case you recieve a `None` and the `milliseconds` converts your `Long` to a duration object that can be understood by Akka.

As you can see, I have more boilerplate than the actual length of my configuration key and my default value. The Play configuration system is cool, but it requires too much writing. If you have loads of things to configure in your code, you end up with a quite clutered code.

But rejoice! Scala Magic™ is there to save us from RSI and from code that makes your reviewer want to kill someone. We can first write a function in our scope to do something like this:

```scala
def configMilliseconds(key: String, default: FiniteDuration): FiniteDuration = 
    Play.configuration.getMilliseconds(key).map(_.milliseconds).getOrElse(default)
```

This is already pretty neat, I can now do `configMilliseconds("myapp.notifier.scheduleEvery", 30 seconds)`… Isn't that a lot shorter? And it didn't take any magic yet.

This code still has an issue, it requires to have an implicit play application in scope, also, I find that it doesn't put much emphasis on where the config is coming from. It would be cool to focus on the name of the configuration key, that would make our code even more readable (but that's my own opinion, I understand some might prefer the previous notation). All we have to do, is create a postfix operator for a configuration key, that is, a `String`.

With scala 2.10, this is easy (it's not too hard with scala 2.9 either), we just write an implicit class which provides the conversions for us. Here is one that provides the set of configuration types that I use most:

```scala
import scala.concurrent.duration.FiniteDuration
import scala.concurrent.duration._
import play.api.Application
import collection.JavaConversions._

object ConfigString {
  implicit class ConfigStr(s: String) {
    def configOrElse(default: FiniteDuration)(implicit app: Application): FiniteDuration =
      app.configuration.getMilliseconds(s).map(_ milliseconds).getOrElse(default)

    def configOrElse(default: Long)(implicit app: Application): Long =
      app.configuration.getMilliseconds(s).getOrElse(default)

    def configOrElse(default: Double)(implicit app: Application): Double =
      app.configuration.getDouble(s).getOrElse(default)

    def configOrElse(default: String)(implicit app: Application): String =
      app.configuration.getString(s).getOrElse(default)

    def configOrElse(default: Boolean)(implicit app: Application): Boolean =
      app.configuration.getBoolean(s).getOrElse(default)
      
    def configOrElse(default: Seq[String])(implicit app: Application): Seq[String]  =
      app.configuration.getStringList(s).map(_.toSeq).getOrElse(default)
  }
}
```

I can now just write: `"myapp.notifier.scheduleEvery" configOrElse(30 seconds)`. You can see that I have chosen a verbose name for the postfix function. Some might prefer shorter names, or go for crazy symbols like Scalaz does.

There are a few things to notice in this code:

- instead of relying on the import of `Play.current`, I take an the app as an implicit argument, this makes it a bit more versatile
- all the functions have the same name `configOrElse`, scala figures out on its own what return type I want from the default value I provide.
- all I have to do now to have access to the config postfix functions is to `import ConfigString._` 

So, to schedule my task, before:

```scala

val start = Play.configuration.getMilliseconds("myapp.notifier.startQueuing")
.map(_.milliseconds).getOrElse(2 minutes)
val every = Play.configuration.getMilliseconds("myapp.notifier.queueEvery")
.map(_.milliseconds).getOrElse(1 hour)

Akka.system.scheduler.schedule(start, every)
(NotificationScheduler.schedule)

```

after:

```scala

val start = "myapp.notifier.startQueuing" configOrElse(2 minutes)
val every = "myapp.notifier.synastry.queueEvery" configOrElse(1 hour)
Akka.system(app).scheduler.schedule(start, every)
        (NotificationScheduler.schedule)
```

Finally, there is a [gist][3] for this code if you want to extend it or make it better.

   [1]: http://www.playframework.com/
   [2]: http://www.playframework.com/documentation/2.1.0/Configuration
   [3]: https://gist.github.com/Mortimerp9/5249018
