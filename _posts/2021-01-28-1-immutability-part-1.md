---
layout: post
title: Immutability (part 1)
subtitle: The benefits of immutability in software
tags: [java, immutability, mutability, functional-programming, good-practices]
comments: true
---

One topic that is particular dear to me when it comes to best practices is immutability. I decided to split this article in two parts as it was getting
too much to ingest at once, so here's the first part of this, presenting what immutability is and why it matters. In the [second part](/2021-01-28-2-immutability-part-2), 
we'll find out how to put it into practice.  

# What's immutability?

Immutability is the characteristic of some entity to never change, and be unchangeable. It contrasts with mutability, which is the property of an entity of being able to change.
In software, immutability refers to the guarantee that the state of a given variable, value or class will never change once this entity has been initialized. On the reverse, 
a mutable object can experience multiple state changes during its lifetime. A typical example would be a mutable list that is initialized as empty and gradually filled with data.
An immutable list would be initialized with all its items from the start, and then it would be impossible for the software to add, remove or change any of the items.

Typically, immutability is a stronger concept in high-level languages - particularly functional ones like Haskell, Scala, Clojure etc - while mutations are very common in
 lower level code (e.g. manipulating pointers), imperative languages (C, Ada etc) as well as object-oriented languages (Java, C#). For low-level code, I won't go into details
 but admittedly mutability is more than justified and allows writing high-performance code, with as few abstractions as possible and working close to the machine (abstractions 
 generally have a performance penalty of some sort). For OO languages, I must say I can't find any good reason why mutability should be the norm, and the goal of this post will
  be to convince you that there isn't any.   

# What makes immutability desirable?
## Cognitive simplicity
The best thing about immutability is how easy it makes it to reason about the state of a program. Think about an object instantiated with a given state, and implementing behaviour that
leverages this state. When a stage change occurs, this could affect behaviour. Therefore, in order to reason about the behaviour of the object, you must be aware of potentially all
 the state changes it has experienced until the point you intend to use it. In some cases this can be straightforward, for example if the object was created two lines above, but if it
 traversed many layers of code and each of them potentially modified the state, it's very complicated, and you *will* get it wrong at some point, because you're human and fallible.
 
Now if the object is immutable, you can understand why this makes things simpler: from the moment the object is initialized to the moment when it's used, its state is guaranteed to remain
constant. No matter how many layers of code it traverses, it will always behave consistently. This makes code much easier to write, and to think about. 

## Good for encapsulation and class design
This section is specifically for object-oriented code. Here are the main points I'd like to make about this:
- creating immutable objects allows a class to fully control a certain responsibility in the application. If a class returns mutable objects, any other class is free
 to modify its state, and thus break the encapsulation and single responsibility principles since multiple classes contributed to the state of the same object, which could
 affect its behaviour of the behaviour of objects that use it.
- as a general statement, we should only allow the code to do what it's meant to do. Any operation that is not allowed but technically possible opens the door to bugs. 
  Thus, if an object doesn't need to be modified, there's no good reason to allow it to be modified.

## Thread-safety
Concurrency is hard, that's a claim that not many people would dispute, if any. To be more precise, running things in parallel (multi-threading or multi-processing) is relatively easy
 as long as your units of execution are independent of each other, which means:
- they you don't share mutable resources (e.g. memory address space, disk files etc)
- no unit of execution has to wait for the result of another unit to get started (otherwise you need to consider things like [deadlocks, livelocks and starvation](https://www.geeksforgeeks.org/deadlock-starvation-and-livelock/)). 
This part is irrelevant to our discussion so I'll say no more!

In the case of multiple units of execution sharing mutable resources, you have to deal with the fact that two units might modify the state of the same resource in an incoherent manner,
leaving it in an incorrect state. A simple example would be a program that increments a counter in parallel without proper concurrency defenses:
- initialize counter C to 0
- thread A reads the value of C (i.e. 0)
- thread B reads the value of C (i.e. 0)
- thread A sets C to the value it previously read, incremented by 1 (i.e. 1)
- thread B sets C to the value it previously read, incremented by 1 (i.e. 1)
- C is now 1, while it has been "incremented" twice

In the example above, it may look simple to avoid, but in real applications it gets a lot more complicated. And guess what? This problem is unique to mutable objects. An immutable object
is inherently safe for concurrency because it's impossible for any unit of execution to modify its state.

## Stateful vs stateless application, side-effects vs pure function
Immutability is closely related to the topic of stateful/stateless functions or applications, and to side-effects.

A stateless application is an application that holds no internal state from one request to the next. Stateless applications are usually praised for the following reasons:
- since there is no state, calling a given operation on the application will always yield the same result given the same input. Therefore, if for some reason the application crashes,
it can simply be restarted without data loss.
- it's easy to scale up, since all you have to do is spin up multiple instances of the application and they do not need any form of communication or data sharing between them
  to work properly and produce consistent results
- it decouples the client from the specific server that talks with it. If you have communication such as web sockets between a client and a server, depending on how you communicate via
  this web socket, it might be the case that the client becomes tied to a specific server instance. If the application is stateless, a client can talk to any of the servers and get the same
  response from them.
- fault-tolerance: you can retry any request on another server in case one becomes unresponsive, and get the same response as the first server would have given you
  
It's similar with functions with side-effects and pure function. A pure function, which produces no state change when it's executed, can be executed in parallel by many execution units, 
in any order and any number of times, and always produce the same results. In [Apache Spark](https://spark.apache.org/) (a library for distributed computing), these properties are advantageously
leveraged by serializing pure functions from a driver program to a fleet of slave nodes, and execute it as independent tasks on a large number of inputs. If any node dies it's ok, just 
reassign its tasks to another node and it will complete the work. If a task fails, no problem, it can just be retried and will produce the same result as it would have if it had never failed.
The same is of course not true of functions that have side-effects. 
 
What we see here is the power of immutability not at the level of your code as in previous sections, but at the level of the entire architecture. These architectures yield many benefits
that are either impossible or very hard to obtain without immutability guarantees. Of course some applications can't be stateless (e.g. a bank account), that's why we also have to invent
complicated things like transactions to make things safer, but if you can enjoy the safety and architectural benefits of immutability when mutability isn't strictly required, why wouldn't you?

# Where would mutability still make sense?

There are some cases in which mutability is useful as we said earlier, perhaps even desirable.
For example, a state machine is a useful concept in some applications, and it fits pretty well with mutability (it literally has "state" in its name). I would argue that you can implement
a state machine without mutation, but it's likely going to be in functional style and might not correspond to you or you team's skillset, or used technologies. 

Another example is performance: since it's not possible to change an immutable object, you can be lead to make modified copies of it, which comes with a performance penalty. 
And as we said earlier, low-level code allows extremely fine-grained optimizations, directly writing into specific address of the memory, writing into a registry etc. It gives a level
of control that you simply cannot have with immutable structures. To balance that a bit, I should mention that functional languages like Haskell use the neat properties enabled by
pure functional programming and immutability to write smart compilers that can optimize the code better than an average developer, and more importantly, much more cheaply. Finally, the 
truth is that most applications in the work are not high-performance applications. You'll rarely need that extra microsecond in real life. Most applications care about fractions of seconds,
or a handful of milliseconds for very fast services. There are only a few domains like image processing, bioinformatics, physics simulations etc, where you actually need to optimize to a 
ridiculous degree because one microsecond on an operation executed billions of times can still save you a lot of time and money. For most other applications, simplicity, safety and ease
of development are much more important factors than raw performance.

----

That was quite a bit of information! By now, you're hopefully on board with the idea of using immutable structures as much as you can in your code. But how can we promote this in our code
and what does it mean, in practice, to write immutable code? That's the topic of the [second part](/2021-01-28-2-immutability-part-2) of this post.
    
# References
    
- [Deadlocks, livelocks and starvation](https://www.geeksforgeeks.org/deadlock-starvation-and-livelock/)