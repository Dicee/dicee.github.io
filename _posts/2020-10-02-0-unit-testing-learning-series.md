---
layout: post
title: Unit testing learning series
subtitle: How to write clean, thorough unit tests
tags: [unit-testing-learning-series, java, junit, unit-test, good-practices]
comments: false
---

The first thing I want to start with in this blog is unit testing. The reason for that is that you can only guarantee your software is as good as your tests. Anything that isn't tested, and tested properly, might break at some point.
In fact, even relatively well-tested software can break, as mistakes or edge-cases can always slip through the cracks. Unit tests are the first line of defense against all bugs directly coming from your software (as opposed to integration
issues that can occur during authentication, network calls etc). Therefore, that's also where good software starts. 

There are a couple of things that are important to keep a high quality and healthy test suite, that's what I'll try to tackle 
in this series about unit testing. The code snippets will be using Java and JUnit5, but the concepts that we'll discuss should apply to most languages. You can find the full list of articles in this series using the 
[unit-testing-learning-series](/tags#unit-testing-learning-series) tag.