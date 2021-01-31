---
layout: post
title: Immutability (part 2)
subtitle: Practice what you preach
tags: [java, immutability, mutability, functional-programming, good-practices]
comments: true
---

In the [first part](/2021-01-28-1-immutability-part-1) of our discussion on immutability, we presented the benefits of immutability, both at the level of code and architecture. 
Now that you're hopefully bought with the idea, let's see how we can make this happen. It's hard to talk about this in a language or paradigm-agnostic manner, so I'll mostly focus
on OOP and Java. For other paradigms like functional programming, you'd find that your language has built-in immutable constructs (e.g. [persistent data structures](https://en.wikipedia.org/wiki/Persistent_data_structure))  

# Basics

To enforce immutability, remember the following:

- use immutable collections. 
   - a popular way to do this is to use Guava, which exposes a few immutable collections (e.g. `ImmutableList`, `ImmutableSet` etc)
 If you need to add elements one by one before you can construct the immutable object, you can just use a builder (e.g. `ImmutableList.builder()`) and call `build()` once you're done populating it.
   - for those fortunate enough to use Java 9+, you can directly use `Map#of`, `List#of` etc, which will give you pretty much the same thing as Guava collections, except perhaps that Guava collections
 have extra guarantees like the fact they are always ordered (even the sets and maps respect insertion order, except when they are sorted sets/maps of course), which is a nice property. 
 Also, I'm not aware of any equivalent for immutable collection builders.
- for [POJOs](https://en.wikipedia.org/wiki/Plain_old_Java_object) and generally all classes with attributes:
    - a common choice is to use [Lombok](https://projectlombok.org/) POJOs to generate repetitive code instead of writing it and having it visally clutter the 
    codebase. If you use Lombok, never use `@Data` or `@Setter` unless absolutely needed, use `@Value` and `@With` (or `@Wither` for older Lombok versions)
    - if you use regular Java without such library, simply remember to make *all* your class attributes `final`, unless you have excellent reasons for wanting to reassign them to different valus
- non-final and/or mutable static constants are dangerous, especially if public. Avoid at all costs.    
- do not use `Arrays.asList`. This makes the list structurally unmodifiable (can't add or remove elements) but still mutable (can change existing values). The only exceptions here are where the value is only used within the same method to use some List functionalities, or when making a copy would be very expensive. In this case, you can wrap with {{code}}Collections.unmodifiableList{{/code}}.
- prefer functional style (streams) to loops most of the time, but with good judgement. When collecting the streams, **use immutable collectors** (e.g. `ImmutableList.toImmutableList()`)
- when using streams, don't make any side-effects in your functions (except of course in `forEach`, and perhaps `peek` operations, or things like `Optional#ifPresent`)

## What if I have to use mutable structures? 

If your data structure or class **really** has to be mutable, observe the following rules:

- expose as narrowly as possible, limit the number of places where mutation happens and the distance between them (both logical and physical distance)
- if an unmodifiable view is enough to pass to other pieces of code that need access to a mutable member of a class, do this (e.g. `Collections#unmodifiableList`)
- be careful about possible race conditions

If you keep things as local as possible and are careful about thread-safety and encapsulation, you should be able to cope just fine with a little bit of mutation in your code.

# Going further
I'll admit straight away, I haven't used nor deeply evaluated most of the libraries I'm going to present below. I have used Vavr in actual projects, and for the other ones I just
read enough about their doc and code to find them valuable to share for your interest, This disclaimer made, I can proceed to present you some Java libraries that make it easier
to use immutability in your code, make your code more function or extend the capabilities of Java streams in a way that gets them closer to what you would get from a
 more functional language like Scala (Haskell purists would say that Scala isn't really functional, and relatively speaking they would be right, but I know Scala much better).

## Immutables
[Immutables](https://immutables.github.io) is sort of Lombok on steroids, and specifically geared for immutable objects. Like Lombok, it's based on annotation processing, and it has 
[*a lot* of features](https://immutables.github.io/immutable.html), among which:
- generating POJOs, along with contructors and mutable builders
- strict builders, which enforce that attributes can only be set once (checked at runtime). This means you can confidently pass the builder to another method or class, knowing 
  that the values you already set will never be overriden (it will throw an exception and likely crash your code though). 
- staged builders, which only allow to set one field at a time and then return a builder of a different type with the next field to set. 
  This ensures that all fields are initialized (which is easy to miss with a regular builder) at compile-time.
- special handling of collection attributes to use immutable collections for those attributes
- lazy attributes, and caching of lazily computed values (like `lazy val` in Scala)
- validation checks and data normalization upon construction
- copy methods
- singletons
- and so on!

To be honest, I find it almost too advanced in some cases. I'm not particularly a fan of code that contains more annotations than actual logic... But other than that, it has a really solid
feature set, and if that makes it easier for you to use and promote immutable data structures, that's all good!

## Vavr
[Vavr](https://www.vavr.io/) is a library that offers a number of immutable and functional constructs in Java. In particular, it contains:
- immutable collections with rich interfaces, in the manner of Scala. It's more powerful than Guava collections in the sense that it adds a lot of nice functional features
 (e.g. `zip`, `takeWhile`, `sliding` etc, for stream processing), but it comes at the expense of some usability since these collections do not implement the regular `java.util` collection
 interfaces. There are converter methods for those, but I still think this could prove limiting in some cases.
- Extended support for futures, promises and streams. Same approach as for collections, these are nice functional features but took the party of not implementing built-in Java interfaces
  to gain more flexibility in their API.
- Classic functional monads such as `Try` and `Either`. It can be super elegant in code. For example, I once used `Either` to  migrate a codebase from using a given serialization model
  to another one, e.g. `Either<Model.V1, Model.V2>`. This means that developers explicitly had to consider both possible models in every operation they were making on the raw bytes, and
  it was impossible to get it wrong (similar to the philosophy of using `Optional` instead of nullable variables). I also love `Try` for storing the result of an operation that can fail 
  and hand it over to a library method that can generically handle a result that potentially failed to be computed.
  
Overall I'd say Vavr is a very good library, it does what it does well and does not over-complicate things.  

## streamex
[streamex](https://github.com/amaembo/streamex) is a library specifically aiming at extending Java streams. It takes a different approach as Vavr since it implements the actual `java.util.Stream`
interface, and is thus fully inter-operable with any Java code that works on streams. This is a good property to have for a language extension like this one.

Compared to Vavr, I'd say it's a less ambitious library, but if you're looking to simplify your streaming code without making too big of a paradigm or programming style change, it's a 
good candidate.

## jOOL
[jOOL](https://github.com/jOOQ/jOOL) is a library that mostly focuses on extending Java streams (same strategy as streamex), as well as a few other functional elements lacking in Java 
like tuples, functions with a large number of parameters, functions that throw exceptions (not nicely supported by built-in Java functional interfaces) etc.

The feature set is quite similar, maybe a bit more extended, than streamex. If you're interested in such stream extensions, I would recommend you to evaluate both and see which ones fits 
your needs and style better. I didn't look at them deep enough to have an opinion myself.

----

Alright, that's enough about this. If this made you curious about functional programming, I can suggest you a few languages to check out:
- **Scala:** a JVM language with many similarities with Java which attempts to fuse functional and OOP styles.
- **Haskell:** probably one of the most functional languages in existence that's also popular. If you're looking for a pure form of functional programming, this is a safe bet.
- **Clojure:** a Lisp dialect on the JVM. Of the three languages I'm suggesting you to read about, this is likely the one that looks the least to anything you've done so far. It's a
  great mind opener to see what other programming styles are out there and how different can look and feel.  
  
Enjoy!
   
# References
- [Persistent data structures](https://en.wikipedia.org/wiki/Persistent_data_structure), very interesting topic on how to implement immutable data structures that make optimal use of memory.
  This Wikipedia page itself is pretty light, but it can give you pointer for more, like for example [persistent hashed array mapped tries](https://en.wikipedia.org/wiki/Persistent_data_structure#Persistent_hash_array_mapped_trie), 
  which get pretty sophisticated.   
- [Project Lombok](https://projectlombok.org/)
- Functional libraries:
  - [Immutables](https://immutables.github.io/immutable.html)
  - [Vavr](https://www.vavr.io/)
  - [streamex](https://github.com/amaembo/streamex)
  - [jOOL](https://github.com/jOOQ/jOOL)