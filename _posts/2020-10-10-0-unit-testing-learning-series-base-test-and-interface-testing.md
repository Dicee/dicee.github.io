---
layout: post
title: Unit testing learning series - Base test class and interface testing
subtitle: Another way of sharing test code
tags: [unit-testing-learning-series, java, junit, unit-test, inheritance, good-practices]
comments: true
---

*This article is part of the [Unit testing learning series](/2020-10-02-0-unit-testing-learning-series).*

---

In comparison to the previous article about matchers and assertions, this one is going to be relatively short. I believe it covers less common use cases, but 
interesting ones nonetheless. Here we're going to talk about a few cases where creating base test classes and inheriting from them can be beneficial.

### A word about inheritance

Just a warning, I personally am not a big proponent of inheritance. This will be the topic of a separate article outside of the unit testing learning series,
but what's important to keep in mind for this post is that I don't advise over-using the patterns I am about to discuss. I will clearly specify in which conditions I think
they do well, but for other use cases it would be up to your judgement.

### What's a base test class and when to use it

Just quickly to make sure we're talking about the same thing: a base test class would be a class that contains logic to prepare a test, cleanup after a test, or implement
generic tests. It doesn't test a specific class of your application, but instead should be sub-classed to create tests that do apply to a specific class. There are other ways
to cleanly encapsulate test setup or cleanup logic: in JUnit4, rules are very handy (see [here](https://www.baeldung.com/junit-4-rules), I won't talk about it myself as Junit5
removed them) and the equivalent in JUnit5 would be extensions. I haven't implemented any JUnit5 extension so far, but here are a few examples of the rules I implemented or used
when using JUnit4:
- a rule allowing to set Joda time to a specific frozen time, and reset Joda to use the real time after the test
- a rule to snapshot environment variables and revert all side effects that were applied to it once the test is done

For this kind of things, you'd probably not use a base test class because these are features that can be useful for a wide variety of tests. A base test class should generally be used
when the sub-classes you intend to have are more coupled than that. Here are a few rules of thumbs:
- you intend to have many tests share the same data or mocks, that you first have to setup
- you have many tests that follow the same pattern, if only you extract some parameters
- you have classes implementing the same interface that have to abide by some properties common to all implementations of this interface, plus some properties specific to the implementation

Let's discuss these cases in more details.

#### Sharing setup/cleanup, data and mocks

It can happen that some tests require a large amount of setup (dummy data and mocks), or at least setup that is not straightforward. If many tests can benefit from the exact same setup,
or share parts of it, you can do as follows:

```java
@MockitoSettings
public abstract class BaseAirplaneTest {
    protected static final String AIRCRAFT_ID = "id";
    protected static final DateTime TAKE_OFF_TIME = new DateTime(2020, 1, 1, 0, 0, UTC);
    protected static final HAE FLIGHT_ALTITUDE = HAE.fromMeters(5000);
    protected static final WeatherConditions WEATHER_CONDITIONS = WeatherConditions.builder()
        .wind(new WindConditions(Speed.fromMeters(17), Angle.fromDegrees(75)))
        .gust(new WindConditions(Speed.fromMeters(50), Angle.fromDegrees(67)))
        .visibility(new Visibility(Distance.fromKilometers(5)))
        .pressure(Pressure.fromPascals(54019))
        .temperature(Temperature.fromCelsius(15))
        /* etc */
        .build();
    
    protected static final City FROM_CITY = City.NEW_YORK;
    protected static final City TO_CITY = City.PARIS;

    @Mock private WeatherSensor weatherSensor;
    @Mock private GpsDevice gpsDevice;

    @BeforeEach
    public static void commonSetUp() {
        // just one example because I'm out of imagination, but you could have various
        // attributes and mocks to setup here
        when(weatherSensor.getMeasurements()).thenReturn(WEATHER_CONDITIONS);
    }
    
    @AfterEach
    public static void commonCleanUp() {
        // same, I'm lazy to find examples here, but you could need to cleanup some resources
        // for all tests extending this one
    }
}
```

Note that none of this *requires* a base class: 
- for the shared data, you could just have a utility class with constants
- for some setup and cleanup steps, you might use rules or extensions
- for mocks, this could be encapsulated in one or more utility classes

However, the combination of all that in a single base test class is still a powerful way to share code with a simple `extends BaseAirplaneTest`. You can also
add methods on the abstract class, for example to factor out the code of common assertions, or any useful helper that acts on the mocks or internal data of the test.

### Interface testing

A perhaps more interesting though rarer example of application is interface testing. By that, I mean the fact of testing common properties of all implementations of a given 
interface in a single class, to ensure all of them match the basic contract, without duplicating any code. Here's a simple example of that:

```java
public interface Animal {
    void drink(Drink source);
    boolean isHydrated();
    void sleep(Duration duration);
    Stamina getStamina();
}

public interface Dog extends Animal {
    void fetch(Ball ball);
    void sit();
}

public abstract class BaseAnimalTest {
    private static final Stamina INITIAL_STAMINA = Stamina.ofPercent(80);

    private Animal animal;
    
    @BeforeEach
    public void setUp() {
        animal = initAnimal(false, INITIAL_STAMINA);
    }
    
    protected abstract Animal initAnimal(boolean hydrated, Stamina stamina);

    @Test
    public void testDrinkAndIsHydrated_acceptsWater() {
        assertFalse(animal.isHydrated());
        animal.drink(Drink.WATER);
        assertTrue(animal.isHydrated());
    }
    
    @Test
    public void testSleepAndGetStamina() {
        assertThat(animal.getStamina(), equalTo(INITIAL_STAMINA));
        animal.sleep(Duration.ofHours(5));
        assertThat(animal.getStamina(), greaterThan(INITIAL_STAMINA));
    }
}

public class DogTest extends BaseAnimalTest {
    @Override
    protected abstract Animal initAnimal(boolean hydrated, Stamina stamina) {
        return new Dog(Race.LABRADOR).withHydrated(hydrated).withStamina(stamina);
    }
    
    @Test
    public void testFetch() {
        // etc
    }
    
    @Test
    public void testSit() {
        // etc
    }
}
``` 

In this case, `DogTest` is going to run all tests from `BaseAnimalTest`, but also the tests in `DogTest`. This can be a powerful way to ensure all implementations
of an interface have a test for all use cases and edge cases that intrinsically apply to the interface, regardless of the implementation.

That's all I had to share about it, see you on the next post which will be a short discussion about test coverage and related tooling.