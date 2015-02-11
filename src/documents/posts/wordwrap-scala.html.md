---
layout: 'content'
title: 'WordWrap with Scala'
---


This is a simple issue: take a `String` and add line breaks after a certain amount of "fill", usually this is to make the string display nicer for humans, cutting at around 78 characters.

[Langref](http://langref.org/scala/strings/printing-strings/string-wrap) has a few solutions, but they cut words, so you end up with something like: 
>The quick brown fox jumps over the lazy dog. The quick brown fox jumps over t 
> he lazy dog. The quick brown fox jumps over the lazy dog. The quick brown fox"

Wouldn't it be nicer if you could cut it around word delimiters?

>The quick brown fox jumps over the lazy dog. The quick brown fox jumps over the
>lazy dog. The quick brown fox jumps over the lazy dog. The quick brown fox"

A simple enough solution is to use a regexp: 

```scala
val wrapRegEx = """(.{1,78})\s""".r
def wrapLine(s: String) = wrapRegEx.replaceAllIn(s, m => m.group(1)+"\n")
```
