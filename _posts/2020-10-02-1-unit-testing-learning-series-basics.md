---
layout: post
title: Unit testing learning series - Basics
subtitle: Let's start with the basics
tags: [unit-testing-learning-series, java, junit, unit-test, good-practices]
comments: true
---

*This article is part of the [Unit testing learning series](/2020-10-02-0-unit-testing-learning-series).*

---

In this article, which is the first in the series, we'll go over the most basic guidelines for writing good unit tests.

### Never be the first offender
The most important thing with a unit test is to write it in the first place. It sounds obvious, but what I'm talking about here is a psychological effect that can have bad consequences on
code quality within a team. It can be referred to as the [broken window effect](https://en.wikipedia.org/wiki/Broken_windows_theory). Applied to software, the idea is that if developers in
a team come across a lot of low quality code during their job, it is more likely that the code they write themselves will be of poorer quality than what they would otherwise be capable of.
Indeed, what is considered acceptable or not, even though there might be objective arguments for or against certain practices, is largely determined by the culture of the team. People will
be less motivated to apply themselves if all they see is bad code, or missing tests. Why should they bother? Nobody will recognize their hard work and everybody else will continue to develop
with a lower bar.

To avoid ending up in this situation, the least you can do is to *never be the first offender*, that is, holding yourself to the highest bar. So don't break that first window, and create a
test class for every new class or functionality you introduce. This will increase the probability that the next person making a change in this class will write a test too.

### Write the code you want your team to write
 
Of course, this initial test should also be written with all the good practices we'll talk about in the series, because a large part of the code is written by imitation of existing code. 
That's what I call *writing the code you want others to write.* The more people will see certain patterns in the codebase, or be reminded of it in a code review, the more likely they will
follow the pattern the next time it is relevant, and so on until the pattern becomes a norm.

### Test content and structure 

- **All happy and non-happy cases should be tested.**
- Tests should typically have the following structure (following this pattern increases consistency, and makes it easy to grasp a test):
   - construct inputs and setup mocks
   - call public API of the class to be tested
   - if the API returns a value, compare this full value with an expected result. If the API is a void method, some mock verification may be done to verify side-effects that were expected
    did occur.
- Tests should typically **NOT** do the following:
   - verify methods were called on mocks for operations which have no side-effect (this would mean testing some implementation details rather than the high-level behaviour)
   - verify that methods were called in a certain order if this order is not relevant to the high-level behaviour (it's almost never relevant!)
   - expose private members or methods as package-private for the purpose of testing. This is generally done because the class to test is not testable, which is a design flaw in itself.
    Testing internals doesn't guarantee the behaviour of the public API (which is what matters to production code) since there is no guarantee these internals are ever called in 
    production code paths, or that they're called correctly. Constructors are an exception to this rule (as long as all constructors of the class call another constructor and there's 
    only one root constructor) as they help injecting stubs.
 - **Avoid duplication.** A good test case is readable and short. I'll probably make a separate article about duplication to explain why it's **evil**. In tests, it's generally not as 
  severe as in production code but still has a significant impact on maintainability. Don't hesitate using a lot of helpers at the bottom of your class to create dummy objects, or factor
  out groups of assertions you make in multiple test cases. What you want to avoid is drowning the information that is relevant to the test in overly verbose code. Also use a setup 
  method to initialize shared resources if required. Some examples below:
 
 **Setup method example**
 ```java
// shared setup. Can be variables, mocks, state on disk etc.
// Especially do this to setup the object under test, having as few calls
// to the constructor as possible increases maintainability and makes it
// easy to navigate through code (usage searches not polluted by test usages).
@Mock private S3Client s3Client;

@BeforeEach
public void setUp() throws IOException {
    fakeXmlFile = temporaryFolder.newFile("output.xml");
    Files.write(fakeXmlFile.toPath(), ImmutableList.of(FAKE_XML_CONTENT));
    s3Uploader = new S3Uploader(BUCKET, PREFIX, s3Client, CLOCK);
}
```

**Example of assertion helper**
```java
// assert helper factoring out repetitive assertion logic
private void assertMeasurementDeltaIs(long newCount, long oldCount, Duration newTime, Duration oldTime) {
    long deltaCount = newCount - oldCount;
    Duration deltaTime = newTime.minus(oldTime);
    assertCorrectMeasurements(deltaTime.dividedBy(deltaCount), deltaCount);
}

private void assertCorrectMeasurements(Duration expectedAvgCollectionTime, long expectedRecentCollectionsCount) {
    assertThat(gcMonitor.averageCollectionTime(), equalTo(Optional.ofNullable(expectedAvgCollectionTime)));
    assertThat(gcMonitor.recentCollectionsCount(), equalTo(expectedRecentCollectionsCount));
}
```

**Example of shared test case template**
```java
// those three tests become trivially small by sharing a test template, and 
// dozen of those can easily be added afterwards. If the base method was duplicated,
// the test would be gigantic and each test case would tune the behaviour in slightly
// different ways, making safe refactors extremely difficult.
@Test
public void testErrorTranslation_runtimeException() {
    assertExceptionCorrectlyHandled(
        new IllegalArgumentException(MESSAGE), 
        InternalServerError.class,
        Level.ERROR
    );
}

@Test
public void testErrorTranslation_awsException_500() {
    assertExceptionCorrectlyHandled(
        newAwsSdkException(MESSAGE, StatusCode.INTERNAL_SERVER_ERROR), 
        DependencyException.class, 
        Level.WARN
    );
}

@Test
public void testErrorTranslation_authException() {
    // other tests use a default value for the published error metric,
    // but here the system customizes the name of the published metric 
    // for authentication failures
    assertExceptionCorrectlyHandled(
        new UnauthorizedException(MESSAGE),
         InternalServerError.class,
         Level.ERROR, 
         "auth.errors.count"
    );
}


private void assertExceptionCorrectlyHandled(
        RuntimeException originalException, 
        Class<? extends RuntimeException> expectedExceptionType,
        Level expectedLogLevel) {
    assertExceptionCorrectlyHandled(originalException, expectedExceptionType, 
        expectedLogLevel, originalException.getClass().getSimpleName());
}

// this tests that GetDocumentRequestHandler.handle correctly translates exceptions 
// from DocumentStore.getDocument in order to generate accurate exceptions and status 
// codes to customers. It also checks that specific test was logged out with the correct
// log level, and   that the correct metrics were published. 
private void assertExceptionCorrectlyHandled(
        RuntimeException originalException, 
        Class<? extends RuntimeException> expectedExceptionType,
        Level expectedLogLevel, 
        String expectedMetricName) {

    when(documentStore.getDocument(DOCUMENT_ID)).thenThrow(originalException);
    Exception e = assertThrows(expectedExceptionType, 
        () -> requestHandler.handle(newGetDocumentRequest(DOCUMENT_ID)));

    boolean expectWrapped = !originalException.getClass().equals(expectedExceptionType);
    assertThat(e, expectWrapped ? hasCause(is(originalException)) : is(originalException));

    String messagePrefix = expectWrapped ? originalException.getClass().getName() + ": " : "";
    assertThat(e, hasMessage(equalTo(messagePrefix + MESSAGE)));

    String originalExceptionName = originalException.getClass().getSimpleName();
    verify(metricReporter).reportFailure("GetDocument", expectedMetricName);
    verifyNoMoreInteractions(metricReporter);

    assertThat(recordingAppender.getEvents(), equalTo(ImmutableList.of(
            SimpleLogEvent.forClass(GetDocumentRequestHandler.class, INFO, "GetDocument called with parameter foo"),
            SimpleLogEvent.forClass(GetDocumentRequestHandler.class, expectedLogLevel, 
                "GetDocumentRequest failed with " + originalExceptionName + " for input foo")
    )));
}
```

**Example of test helper class**
```java
/**
 * An appender which stores all the events appended to it, in order to test
 * what was logged by a class (example usage in the previous snippet). This
 * type of test utils can be reused in many different tests to make them
 * easier to write and more concise.
 */
public class RecordingAppender extends AbstractAppender {
    public static RecordingAppender attachedToRootLogger() {
        RecordingAppender recordingAppender = new RecordingAppender();
        getRootLogger().addAppender(recordingAppender);
        recordingAppender.start();
        return recordingAppender;
    }

    private final List<SimpleLogEvent> events = new ArrayList<>();

    public RecordingAppender() {
        super("RecordingAppender", null, null);
    }

    @Override
    public void append(LogEvent event) {
        events.add(SimpleLogEvent.from(event));
    }

    public void detachFromRootLogger() {
        getRootLogger().removeAppender(this);
    }

    public List<SimpleLogEvent> getEvents() {
        return unmodifiableList(events);
    }

    private static Logger getRootLogger() {
        return (Logger) LogManager.getRootLogger();
    }

    public void clearEvents() {
        events.clear();
    }
}
```

**Example of dummy object construction helper**
```java
@Test
void testFilterByType() {
    // mocks the underlying document store
    setupDocumentStore(ImmutableList.of(
        newDocument("id1", DocumentType.PHONE_BILL),
        newDocument("id2", DocumentType.QUOTE),
        newDocument("id3", DocumentType.BANK_STATEMENT),
        newDocument("id4", DocumentType.ELECTRICITY_BILL),
        newDocument("id5", DocumentType.PHONE_BILL)
    ));
    assertThat(docStoreClient.filterByType(DocumentType.PHONE_BILL), equalTo(ImmutableList.of(
         newDocument("id1", DocumentType.PHONE_BILL),
         newDocument("id5", DocumentType.PHONE_BILL)
     )));
}

// here there are 5 fields, but we only care to set one of them for the purpose of
// the test. Hence, why should we have to read the rest of the initialization that
// doesn't matter for the test? This method allows tests to be much more concise 
// and focus on the tested functionality.
private static Document newDocument(String documentVersionId, DocumentType documentType) {
    return Document.builder()
            .withId(documentId)
            .withType(documentType)
            .withFormat(DocumentFormat.XML)
            .withPermissionGroup(PermissionGroup.ADMIN)
            .withEncryptionType(EncryptionType.UNENCRYPTED)
            .build();
}
```

Anyway, you get the idea. Anything that can be done to make production clear and readable can and should also be applied
to unit tests. Short, well-factored unit tests exempt of duplication will make it much easier to add new tests and reason about
coverage (if it's difficult to understand what a test does, it's difficult to know what is or isn't tested). There are many more
topics to discuss, but hopefully this first pass will be helpful to some people. 