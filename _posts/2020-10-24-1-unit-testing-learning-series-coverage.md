---
layout: post
title: Unit testing learning series - Test coverage and test case automation
subtitle: A metric you shouldn't brag too hastily about
tags: [unit-testing-learning-series, java, mutation-testing, generative-testing, test-coverage, clojure, haskell, good-practices]
comments: true
---

*This article is part of the [Unit testing learning series](/2020-10-02-0-unit-testing-learning-series).*

---

I'm not a big expert on the topic of test coverage or some advanced forms of testing (mutation testing, generative testing), but I just
wanted to touch on it a bit. Mainly, I want to open your mind to different types of testing in case you never heard of them, as well as share
my view on the use of test coverage as a metric.

# What is test coverage?

Test coverage is a metric, typically expressed as a percentage, that represents the fraction of your code that is covered by at least one test. The most used types of
coverage are line coverage (what percentage of lines in your code were traversed during tests) and branch coverage (same thing for code branches). Two well-known libraries for
calculating coverage for Java codebases are [Jacoco](https://github.com/jacoco/jacoco) and [Cobertura](https://cobertura.github.io/cobertura/). 

I won't go in details, there are plenty of resources readily available with a Google search that will explain this better than me.
For reference, [this article](https://jasonrudolph.com/blog/2008/06/10/a-brief-discussion-of-code-coverage-types/) is pretty good and succinct.

## The fallacy of high coverage

Occasionally, I receive communication from recruiters that boast about the test coverage of the team they're hiring for, which never fails to make me smile. The reason why I dislike 
test coverage is that I've seen it used in detrimental ways on various occasions. The problem with enforcing a level of test coverage is that sometimes, it forces to spend more time
on a commit that was perfectly fine in order to satisfy the coverage rule (this happens in particular in codebases where coverage drops are not allowed, even by a tiny percentage).
I've seen people adding redundant tests or change the way their code was written just to be able to build and ship their code, which wasn't helpful in terms of validating the software, 
and also made the codebase less maintainable.

The second problem I see is that coverage metrics can drive some people to only or mostly care about the number, without acknowledging the flaws of the metric. Indeed, you can have high
line or even branch coverage without having gone through the most crucial edge cases of your service. This is detailed in the article I shared in the previous section. Worse, even with
perfect coverage which does traverse your entire code, if you're not making the right assertions, it's pretty much worthless. It's nice to execute your application's code, but you also
need to make all appropriate validations about the output it returned and the state it modified. Therefore, placing blind trust in coverage metrics is madness to me, and I don't think
strictly enforcing flawed metrics in your build will help your teammates, in particular the most junior ones, to think by themselves in terms of use case/edge case as opposed to just
trying to get coverage up.

## A sane way of using coverage metrics

This is purely my opinion, but I find test coverage beneficial when it is used to inform but not to break builds. At Amazon, we have an internal code review tool that can be hooked up
with automatic code analysis tools. One of them leverages the output of the build to find coverage artifacts generated by some of the most popular coverage libraries. The tool fetches the
coverage report, and uses it to post an automatic comment on the code review that summarizes the state of the coverage in the codebase before and after the commit. It will also discretely but
visibly highlight the lines that have not been covered, which incidentally helps the reviewer identifying what edge cases might be missing in the tests, and ease the review of test classes. 
Additionally, the tool allows to configure a threshold of total code coverage and new code coverage (coverage of the code introduced by the commit). If the commit breaks one of the threshold, you will clearly see it 
in the auto-generated comment. However, this won't prevent you from pushing the code if someone approves it, it's purely informational.

Some teams have an additional hook in their pipeline which can impose strict thresholds and conditions (e.g. no coverage drop from one revision to the next), but that's taking it too far
in my opinion. What I like about the code review tool is that it makes coverage visible, and it relieves you from the task of manually checking the report generated by your test coverage
library. A few times, seeing this comment or the uncovered lines highlight actually prompted me to increase the score, but the fact that at the end of the day you as a developer are 
responsible to determine whether the testing was good enough means that you'll need to think about it carefully and not blindly optimize a metric.

# Exotic testing/test case automation

I call it exotic because I've never actually used these techniques, but I found them interesting enough to spread awareness of their existence.

## Mutation testing

The first time I encountered this, I didn't know this had a name. I basically had a colleague who occasionally enjoyed voluntarily breaking a functionality by making small changes like removing
a line, inverting a condition etc, and then running the tests to see if they still passed. If they did, it meant tests were not robust enough and we should add some.

Of course, doing this manually is time-consuming and you'll likely miss just as much edge cases doing this than by simply reading the code to see which cases should be tested. Well, 
it turns out that there are tools that allow automating this process. One well-known framework to do that in Java is [PITest](http://pitest.org/). I won't go deeper simply because I
don't know much more than this as I never used it yet (maybe a future experiment when I have some time!), I just wanted to broaden your horizons, you can research it more on your own
if you're interested. 

## Generative testing

Another form of testing I've never tried but I heard of is generative testing. Essentially, it's an approach where you declaratively specify how your
code should behave, what your inputs look like etc. Using these declarations, the framework will generate test cases for you and try them out against your code. The avdantage is 
that you can generate a huge amount of test cases with little developer effort. This can be useful if your test space is too large to be explored by traditional
hand-crafted example testing.

The languages that fostered this type of testing are from the functional world (e.g. [QuickCheck](https://hackage.haskell.org/package/QuickCheck) for Haskell,
[test.check](https://clojure.org/guides/test_check_beginner) for Clojure), because of their powerful type capabilities. However, some Java tools also support it, 
like [junit-quickcheck](https://github.com/pholser/junit-quickcheck). 

Note that without using a library, you can still do simple things that resemble generative testing if your use case is not sophisticated. Basically, you could generate
your test's input with some amount of randomness. The main thing I want to point out here is that you should control the seed of the random generator. If you use a random seed,
at least log it out, otherwise you'll never be able to reproduce the issue when a test fails for a very specific input! Additionally, it should be noted that it's often quite
difficult to write good assertions for random inputs, so it's not an appropriate technique for all the things you might want to test. It's mostly useful for contracts that have
clear properties and invariants, or for very large test spaces (e.g. a function that works on a general graph data structure).

# References
- [A brief discussion of code coverage types](https://jasonrudolph.com/blog/2008/06/10/a-brief-discussion-of-code-coverage-types/) (Jason Rudolph)
- [Jacoco](https://github.com/jacoco/jacoco), [Cobertura](https://cobertura.github.io/cobertura/) (coverage calculation libraries)
- [PITest](http://pitest.org/) (mutation testing library)
- [QuickCheck](https://hackage.haskell.org/package/QuickCheck) (generative testing for Haskell)
- [test.check](https://clojure.org/guides/test_check_beginner) (generative testing for Clojure)
- [junit-quickcheck](https://github.com/pholser/junit-quickcheck) (generative testing for Java)