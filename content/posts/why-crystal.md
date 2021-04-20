---
title: "Why Crystal"
date: 2019-02-21T17:39:48+03:00
draft: false
author: Vlad Faust
tags: [Crystal]
---

Crystal is awesome language, but why would you want to start using it right now?

<!--more-->

**Table of contents:**

1. [What is Crystal](#what-is-crystal)
2. [What is Onyx Framework](#what-is-onyx-framework)
3. [Why using Crystal](#why-using-crystal)
4. [Companies using Crystal in production](#companies-using-crystal-in-production)
5. [Common misbeliefs](#common-misbeliefs)

## What is Crystal

[Crystal](https://crystal-lang.org/) is a common-purpose compiled language built on top of [LLVM](https://llvm.org/). Its siblings are Clang, Nil, Kotlin, Julia, Swift, Rust and many more. Thanks to LLVM, Crystal programs fully compile into fast binary code, unlike common interpreted language implementations, for example Ruby [MRI](https://en.wikipedia.org/wiki/Ruby_MRI) or [CPython](https://en.wikipedia.org/wiki/CPython).

## What is Onyx Framework

[Onyx](https://onyxframework.com/) is a common-purpose framework with loosely coupled modules to develop appilications in Crystal, faster. Today it has [HTTP](https://github.com/onyxframework/http), [REST](https://github.com/onyxframework/rest) and [SQL](https://github.com/onyxframework/sql) modules.

## Why using Crystal

- **Save big on servers.** Compiled programs consume much less CPU and RAM resources, mostly because of pre-defined type sizes and compilation-time optimizations (e.g. code inlining)
- **Have less errors in production.** No more unexpected `NoMethodError`, as all calls are checked in compilation-time
- **Make use of modern language features.** Generics, macros, go-like routines and more at your service
- **Use familiar, expressive Ruby-like syntax.** Crystal has a beautiful syntax inspired by Ruby
- **Enjoy incredible documentation.** The [docs](https://crystal-lang.org/reference/) are very thorough and explainatory. Also Crystal is written in Crystal, that's why its [API](https://crystal-lang.org/api/0.27.2/) is easy to read and undestand

## Companies using Crystal in production

The language is not 1.0 yet, but it is already used by many companies in production, including [Manas](https://manas.tech/), [Yahoo! JAPAN](https://www.yahoo.co.jp/), [Nikola Motor Company](https://nikolamotor.com/), [Rainforest QA](https://www.rainforestqa.com/), [Diploid](http://www.diploid.com/), [NeuraLegion](https://www.neuralegion.com/), [Sikoba](http://www.sikoba.com/www/index.html) and [more](https://github.com/crystal-lang/crystal/wiki/Used-in-production).

## Common misbeliefs

However, some developers still doubt on adopting Crystal, usually guided by the following reasons:

### Not 1.0, thus expect many breaking changes

Really breaking changes are rare, as the language is pretty mature nowadays.

Crystal is distributed in many ways, including package repositories and Docker registry, which allows you to effectively lock to a particular language version unless updated. The same applies to [shards](https://github.com/crystal-lang/shards) (these are Crystal packages), as they usually are hosted on GitHub with enforced semantic versioning.

So, regardless of new versions, your application built today will be compile-able for a long time.

### Poor ecosystem

This statement may have been actual years ago. As of today, the language ecosystem is able to cover most of the needs of any application. Check [Awesome Crystal](https://github.com/veelenga/awesome-crystal) to see it with your own eyes.

And if you struggle to find an already existing solution, you can easily implement your own shard or even [bind a C library](https://crystal-lang.org/reference/syntax_and_semantics/c_bindings/).

### Lacks IDE support

Monster IDEs like Intellij Idea do not support Crystal yet, but most of the modern text editors like Sublime Text, Atom and VS Code do.

### No parallelism

In fact, neither Ruby MRI nor CPython have real parallelism as well. Read more about [GIL](https://en.wikipedia.org/wiki/Global_interpreter_lock) on Wikipedia.

Crystal implements [concurrency with lightweight fibers](https://crystal-lang.org/reference/guides/concurrency.html) inspired by Goroutines. And if you really need parallelism, it can be achieved with `Process.fork` method (similar to Ruby's `fork`), which is suitable for most web-applications.

Furthermore, in the real, containerizated, world, it's common to have multiple images of the same single-threaded server process running simultaneously.

Summing it up, the current lack of the parallelism is not really an issue for most of the (web) applications in the wild.

But it also worth noting that parallelism has quite a high priority in the to-do list.

### No Windows support

You can still develop on Windows with [WSL](https://en.wikipedia.org/wiki/Windows_Subsystem_for_Linux). But anyway, a Unix OS is a better match for your server.

It's true that you can't just install-n-go on Windows yet, but it will be possible in the future.

## Conclusion

Even if you still not sure whether to use the language or not, it worth at least trying. Follow the [Building JSON APIs with Onyx](/posts/creating-json-apis-with-onyx-part-1) tutorial to find out how simple and enjoyable it is to create bullet-proof web applications with modern compiled languages!
