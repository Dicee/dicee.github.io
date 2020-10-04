---
layout: post
title: Unit testing learning series - Assertions, mocking, and matchers
subtitle: The art of good assertions
tags: [unit-testing-learning-series, java, junit, unit-test, hamcrest, mockito, mocking, good-practices]
comments: true
---

*This article is part of the [Unit testing learning series](/2020-10-02-0-unit-testing-learning-series).*

---

I'm not going to cover all there is to say about mocking in this article, I think this would require a separate article about writing
testable, injectable code. Here, I just want to focus on a crucial aspect of assertions and mocking that I often see violated. I'm 
talking about the fact that **assertions and mocked behaviour should always be as precise as possible**.

I'll be talking about and showing code snippets using [Hamcrest](http://hamcrest.org/) and [Mockito](https://site.mockito.org/), so you might want to 
read a bit about it if you're not familiar with it.

### Field-by-field comparison vs full value comparison.

Field-by-field comparison is the fact of comparing two objects on a field-by-field basis, typically done with a series of
`assertEquals` (I'll explain later in this article why I **do not** recommend using `assertEquals`), but more generally the following
comment applies to all test assertions that only assert a specific property about a 
given object. Comparing field-by-field has the disadvantage that when a new field is added, nothing forces the developer to update the
tests to add a comparison for this field. Therefore, the test could be silently broken. Plus, it's very verbose. In contrast, full 
value comparison is the fact of comparing an actual object to an expected one simply using a single equality assertion on the whole
object.

**Examples**
```java
// if we keep only this first assertion the list might have the right size but 
// contain garbage
assertEquals(list.size(), 3);
// that's a bit better, now we're also testing the actual values stored in the list
assertEquals(list.get(0), 1);
assertEquals(list.get(1), 2);
// oops, the developer forgot to test the third element! It could be wrong, the test 
// would still pass
 
// here, it's impossible to forget anything. We fully describe the expectations we 
// have on the list because there's nothing else to be said about a list than what 
// it contains.
assertThat(list, equalTo(ImmutableList.of(1, 2, 3)));
 
// -------------------------------- //
 
// what if the associated value is incorrect? What if the map rightly contains 
// this key but also contains keys it should not contain? Also if this fails,
// it's annoying to debug because JUnit won't print the actual value, so we'll
// need to add logs or run the debugger to know what's going on.
assertTrue(map.containsKey("hello"));
// with this, we know "hello" is the only key in the map and we also validate the
// associated value is correct. If the test fails, we'll have a clear string diff
// between the actual result and the expectation.
assertThat(map, equalTo(ImmutableMap.of("hello", "world"));
```

### Guidelines for good mocking

When it comes to mocking behaviour, the same principles apply: you want your mocks to be as precise as possible. I see many people
using the `any()` argument matcher in Mockito, which means the test basically doesn't validate at all how the mocked class was called
in order to trigger the mocked behaviour. A good mock does not only allow to provide a desired output that the tested class requires
to implement its functionality, it also validates that the correct inputs were provided to the mock. In production code, the provided
input would obviously impact the output, therefore your mocks should also care about their inputs. As everything, there will be exceptions
to this, but the default guideline is to start with the most accurate mocking/comparison possible, and downgrade to something less specific
only if required.

**Examples**

```java
// bad
when(s3Client.getObjectAsString(anyString(), anyString())).thenReturn(/* fake response */);
assertThat(historyFetcher.getHistory(DocumentClass.HR, "doc1"), equalTo(HISTORY_DOC_1));

// good
String expectedBucket = "hr-documents-prod-us-east-1";
String expectedKey = "history/doc1/summary.txt";
when(s3Client.getObjectAsString(expectedBucket, expectedKey)).thenReturn(/* fake response */);
assertThat(historyFetcher.getHistory(DocumentClass.HR, "doc1"), equalTo(HISTORY_DOC_1));
``` 

In the example above, the first test will pass even if `HistoryFetcher` selects the wrong bucket and/or the wrong key within this bucket.
This is obviously a terrible test, as it leaves a gigantic amount of room for undetected bugs. In contrast, the second test is 100% specific about
the expectations, and any deviation of the behavior of the class compared to these expectations will cause the test to fail. The same thing applies to
verifications on mocks, e.g. in Mockito, `verify(myMock).myMethod(param1, param2)` allows to verify a method was called on a mock with given parameters. 
In this case too, it's super important to make parameters as specific as possible, and not use `any()` just because it's less work to do so.

#### Don't mock data classes

A more minor note about mocking, but interesting too. There are some classes that you probably almost never want to mock, and instead want to use "the real thing". Indeed, a
mock will always be a less faithful simulation of a given class than the class itself. If a mock was a perfect simulation of a class, then it would be as complex as the class
to setup, and you could just use the class itself at this point. That's what you should do for data classes (i.e. [POJOs](https://en.wikipedia.org/wiki/Plain_old_Java_object),
but potentially with a few simple methods that directly act on the data contained by the object). 

There could be a few exceptions for objects that are complex to create, but a lot of the time mocking data is just a maintenance burden:
- methods of the class might use the same data internally, and thus must be kept in sync if mocked, in order to emulate realistic behaviour
- in general a lot more verbose to mock 3 getters than calling one constructor
- when mocked, adding new fields or using fields not previously used by the tested class can break tests at runtime but not compile-time (missing mocked getter). 
  This makes it really painful to update the tests as you have to run the full build and its tests to figure out which tests need to be changed. In contrast, if
  you simply used the data classes directly, all you'll need to do is compile the code or find all the usages of the constructor to know every call site that needs
  to be updated. Plus, that's a much more trivial update to perform than fixing a test with missing mocked values, which can cause non-obvious errors if `null` is a valid value
  for the getter, since the tested code will simply continue executing and explore unexpected branches.  
  
#### Use strict mocks
 
 In Mockito, strict mocks are mocks that will fail the test if they detect that they might have been misused in the test. For example, strict stubs won't allow you to mock methods 
 that the test doesn't actually use, or will fail the test before it ends if the parameters of a mocked call are different from the expectation. In short, strict stubs allow you
 to make a cleaner and more correct use of mocking, and help you make your tests more reliable. For more information, you can start [here](http://blog.mockito.org/2017/01/clean-tests-produce-clean-code-strict.html).
 
 Here's how you'd use strict stubs in Mockito with JUnit5:
 
 ```java
// the default setting is Strictness.STRICT_STUBS if you do not specific anything.
// Otherwise, you can specify a looser strictness, but only if you have *very* good
// reasons. For JUnit4, use @RunWith(MockitoJUnitRunner.StrictStubs.class).
@MockitoSettings // https://javadoc.io/static/org.mockito/mockito-junit-jupiter/3.2.4/org/mockito/junit/jupiter/MockitoSettings.html
public class MyTest {
    // will be automatically instantiated and configured as a strict mock
    @Mock Function<String, String> mock;
}
```
    
### Use matchers    
    
Pure equality isn't always possible to guarantee in a test (e.g. comparing different types of input streams with the same data), so sometimes we'll need to make our assertions using
matchers. Matchers can be useful either to simplify an assertion that would otherwise take several lines to setup, or to implement a slightly less rigorous equality when exact equality
cannot be achieved. I'll talk about it in a minute, but for now I just want to convince you to use matchers in all of your assertions, including those where pure equality is possible.

#### assertEquals vs matchers

`assertEquals` sucks, and I'm going to prove it to you:
- lots of testing frameworks are designed to sound as fluent as possible, just like natural language. There's a good reason for that: a test is easier to understand when it is structured like a specification document. Matchers help writing more fluent tests.
- matchers are more powerful than `assertEquals`, `assertTrue` etc, which are very basic testing constructs. They can do the same thing in a much more concise way, while generating better error messages and being more readable. See examples below.
- with `assertEquals`, you can easily use the wrong parameter order and thus generate incorrect error messages when the test fails, which can be confusing for the person debugging. `assertThat(actual, equalTo(expected))` is much clearer because the order is implied by the structure of the "sentence".
- to see an example of a (more-than-average) complex custom matcher, see below. For classes that are used a lot across the codebase and which instances are not easy to compare with each other, this can be tremendously useful and elegant.
    
**Examples**        
 ```java
assertTrue(map.isEmpty()); // doesn't print anything useful when it fails
assertThat(map, is(empty())); // prints the map in the error message when it fails
 
// the hash set allows comparing elements of a list regardless of their order, but 
// it's verbose and doesn't support duplicates
assertEquals(new HashSet<>(actualList), new HashSet<>(expectedList)); 
// much better! The matcher will generate a message that lets you know exactly which
// elements were found that shouldn't be there or which elements were missing
assertThat(actualList, containsInAnyOrder(expectedList.toArray())); 
 
// ... and much more built-in matchers, plus custom ones you could write when it makes sense to do so
```

To be clear, it's ok to use `assertTrue` and `assertFalse` when what you're testing is truly a boolean property, for example `assertTrue(number.isPrime())`. Here the tested method returns a boolean
already, so it makes sense to make a boolean assertion.
    
#### Custom matchers    

In some cases, you'll benefit from defining your own matchers. I've had the case in one of the teams I've worked in, where we had a large amount of geometry logic. This can be tough to test
due to rounding errors, so that's one case where exact equality isn't generally possible. You can do your best to select inputs that are going to generate integer results for all geometry operations,
but sometimes the geometry is just too complicated to allow this. 

As a result, that's the kind of code I was seeing a lot in our codebase:

```java
assertEquals(expectedLat, position.getLat(), precision);
assertEquals(expectedLon, position.getLon(), precision);
assertEquals(expectedAltitude, position.getAltitude(), precision);
```   

If you've been attentive in the previous paragraphs, you'll know that seeing this made my eyes bleed as this is field-by-field comparison, and uses `assertEquals`. This specific example seems benign because
there is little chance that a class named `Point3D` gets new fields added to it, so you could just stick these assertions in a method and reuse it in all your tests. It wouldn't be too bad,
but the error messages wouldn't be great (you only see the part that failed, not the entire value, which can give hints of what went wrong) and for more complex classes like polygons, which
contain a lot of points themselves, it just gets impractical.

What you can do in this case is define a custom matcher that will allow you to write this kind of code:

```java
assertThat(position, is(closeTo(expectedPosition, precision)));

Polygon expectedPolygon = Polygon.closed(ImmutableList.of(
    new Point3D(...), 
    new Point3D(...), 
    /* etc */
));
assertThat(polygon, is(closeTo(expectedPolygon, precision)));
```

Beautiful, isn't it? Below is an example of custom Hamcrest matcher for a `Point3D` class:
       
```java
public static class Point3DMatcher extends TypeSafeMatcher<Point3D> {
    private final Point3D expected;
    private final double precision;
    private final Matcher<Double> latMatcher, lonMatcher, altitudeMatcher;

    public static Point3DMatcher closeTo(Point3D expected, double precision) {
        return new Point3DMatcher(expected, precision);
    }
    
    private Point3DMatcher(Point3D expected, double precision) {
        this.expected = expected;
        this.precision = precision;
        this.latMatcher = Matchers.closeTo(expected.getLat(), precision);
        this.lonMatcher = Matchers.closeTo(expected.getLon(), precision);
        this.altitudeMatcher = Matchers.closeTo(expected.getAltitude(), precision);
    }

    @Override
    public void describeTo(Description description) {
        description.appendText(String.format(
            "A Point3D within <%s> field-by-field precision of %s", 
            precision, expected
        ));
    }

    @Override
    protected boolean matchesSafely(Point3D actual) {
        return latMatcher.matches(actual.getLat()) && 
                lonMatcher.matches(actual.getLon()) && 
                altitudeMatcher.matches(actual.getAltitude());
    }

    @Override
    protected void describeMismatchSafely(Point3D actual, Description description) {
        description.appendText("was " + actual);
    }
}
```

In real life, that's not exactly what we did. We had so many of these matchers that we created a `CompositeMatcher` allowing to construct new matchers just like this:

```java
public static Matcher<Point3D> closeTo(final Point3D expected, final double epsilon) {
    return CompositeMatcher.<Point3D>builder()
            .matchItem(MatchedProperty.compareWith(expected, "latitude", Point3D::getLat, closeToDouble(epsilon)))
            .matchItem(MatchedProperty.compareWith(expected, "longitude", Point3D::getLon, closeToDouble(epsilon)))
            .matchItem(MatchedProperty.compareWith(expected, "altitude", Point3D::getAltitude, closeToDouble(epsilon)))
            .expected(expected)
            .build();
}

public static Matcher<Speed> greaterThan(Speed expected) {
    return CompositeMatcher.<Speed>builder()
            .matchItem(MatchedProperty.compareWith(expected, "metersPerSecond", Speed::getMetersPerSecond, Matchers::greaterThan))
            .expected(expected)
            .build();
}

private static Function<Double, Matcher<? super Double>> closeToDouble(double epsilon) {
    return d -> Matchers.closeTo(d, epsilon);
}

// ... and many more
```

I won't share this as an example, if you're interested in building a nice `CompositeMatcher`, it can be a good exercise. Regarding mocks, Mockito also allows 
writing custom matchers to use both for mocking behavior and making verifications. The interface to implement would be `org.mockito.ArgumentMatcher`.

That closes this chapter about assertions, I hope this convinced you to give you and your team all the tools to allow your assertions and mocks to be as specific as possible.