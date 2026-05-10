---
layout: post
title: "Best practices series - part 2: service wrappers"
subtitle: Wrap'em up!
tags: [best-practices-series, java, micro-services, good-practices]
comments: true
---

*This article is part of the [best-practices-series](/tags#best-practices-series).*

---

When you go out in winter, do you walk around naked in the streets, flaunting whatever attributes nature endowed you with to all passerby? No? Well, it's the same with your service dependencies.
Ok maybe not quite the same, but I thought it was a catchy introduction.

First, let's clarify what I mean by a (service) dependency. If service A calls service B, then B is a dependency of A. I'm thus talking about dependencies in your architecture such as APIs, databases etc, not libraries.
Generally, a service dependency executes in a different environment than your service: different container at least, or entirely different infrastructure. Any time you have such a setup, you will need to make network calls
to reach this dependency. There are two main ways it could happen: synchronously and asynchronously. I'm not talking about whether you are using a sync or async programming paradigm. If an API is synchronous, you could still
locally call it asynchronously by using multi-threading or some other concurrency feature of your language. I am talking about the protocol used by the service API to communicate with yours. A typical HTTP(S) endpoint is synchronous:
you send a request to it, and some time later, in general not too long, you get a response. This is the main case I want to cover. For asynchronous contracts (for example, pushing a message to a queue), most of what I am going to cover here
is not very relevant.

In a complex architecture, it is likely that each of your services depends on multiple other services. This article will discuss the best practice to write "wrappers", or "adapters", around the client of these dependencies.

### Why should dependency clients be wrapped?

There are numerous advantages to doing this:
- possibility to add standard logging, metrics or error handling: any of your dependencies can impact the health, performance and latency of your service. As such, it is useful
  to collect metrics for each of them. If those metrics are standardized - for those that are cross-functional - this makes things even better.
- simplifying the interface of the service for the rest of your classes, sometimes adding higher-level functions that build on top of the external API (for example converting a batch list API into a stream of list results for other classes to consume it without having to know how the list calls are actually batched)
- encapsulating business knowledge about the dependency (for example, the intricacies of its API)
- not always, but can be useful to convert the external data model to the internal one. It can help isolating your business logic from the data model of the dependencies and make migration easier if you swap between two contract versions of the same service, or migrate to another service entirely. 
- provides a single place to change the way all calls to the external sites are made, rather than having to refactor all call sites. For example, in the case of a new parameter being added to the API.

### How should they be wrapped? 

I'll start with a quick example of service wrapper for S3.

```java
@Log4j2
@RequiredArgsConstructor
public class S3Client {
    private final software.amazon.awssdk.services.s3.S3Client s3;

    public HeadObjectResponse headObject(String bucket, String key) {
        return getFromS3(bucket, key, (b, k) -> s3.headObject(builder -> builder.bucket(b).key(k)), "object metadata");
    }

    public InputStream getObjectStream(String bucket, String key) {
        return getFromS3(bucket, key, (b, k) -> s3.getObject(builder -> builder.bucket(b).key(k)), "object stream");
    }

    public String getObjectAsString(String bucket, String key) {
        return getFromS3(bucket, key, (b, k) -> s3.getObjectAsBytes(builder -> builder.bucket(b).key(k)).asUtf8String(), "object");
    }

    public void getObjectToFile(String bucket, String key, Path destination) {
        try (InputStream inputStream = getObjectStream(bucket, key)) {
            Files.copy(inputStream, destination);
        } catch (IOException e) {
            throw new UncheckedIOException(e);
        }
    }

    /// Eagerly lists all the objects in a given bucket under a certain prefix. May run multiple batch requests.
    public List<S3Object> listObjects(String bucket, String prefix) {
        return streamListObjects(bucket, prefix).toList();
    }

    /// Lazily lists all the objects in a given bucket under a certain prefix. May run multiple batch requests.
    public Stream<S3Object> streamListObjects(String bucket, String prefix) {
        return AwsErrorHandling.decorate(
                () -> "Failed to list bucket %s with prefix %s".formatted(bucket, prefix),
                () -> s3.listObjectsV2Paginator(builder -> builder.bucket(bucket).prefix(prefix)).contents().stream()
        );
    }

    public PutObjectResponse putObject(String bucket, String key, Path source, PutOptions options) {
        return putObjectInternal(bucket, key, RequestBody.fromFile(source), options,
                () -> "Failed to put file %s in S3 for %s/%s".formatted(source.toAbsolutePath(), bucket, key));
    }

    public PutObjectResponse putObject(String bucket, String key, String content, PutOptions options) {
        return putObjectInternal(bucket, key, RequestBody.fromString(content), options,
                () -> "Failed to put object in S3 for %s/%s".formatted(bucket, key));
    }

    private PutObjectResponse putObjectInternal(String bucket, String key, RequestBody body, PutOptions options, Supplier<String> msgSupplier) {
        return AwsErrorHandling.decorate(msgSupplier, () -> {
            PutObjectRequest request = PutObjectRequest.builder()
                    .bucket(bucket)
                    .key(key)
                    .metadata(options.getMetadata())
                    .build();

            Stopwatch stopwatch = Stopwatch.createStarted();
            PutObjectResponse response = s3.putObject(request, body);

            Duration duration = stopwatch.elapsed();
            double bitRate = body.optionalContentLength().get() / 1024.0 / duration.toSeconds();
            log.info("Done uploading %s to bucket %s in %s (%.2f kB/s)".formatted(request.key(), request.bucket(), logFriendlyDuration(duration), bitRate));

            return response;
        });
    }

    public CopyObjectResponse copyObject(CopyObjectRequest request) {
        String srcBucket = request.sourceBucket();
        String srcKey = request.sourceKey();
        String destBucket = request.destinationBucket();
        String destKey = request.destinationKey();

        return AwsErrorHandling.decorate(
                () -> "Failed copying object s3://%s/%s to s3://%s/%s".formatted(srcBucket, srcKey, destBucket, destKey),
                () -> {
                    Stopwatch stopwatch = Stopwatch.createStarted();
                    CopyObjectResponse response = s3.copyObject(request);
                    log.info("Done copying object from s3://{}/{} to s3://{}/{} in {}",
                            srcBucket, srcKey, destBucket, destKey, logFriendlyDuration(stopwatch.elapsed()));
                    return response;
                }
        );
    }

    private static <T> T getFromS3(String bucket, String key, BiFunction<String, String, T> s3Call, String outputDescription) {
        return AwsErrorHandling.decorate(() -> "Failed to get %s from %s/%s".formatted(outputDescription, bucket, key), () -> {
            Stopwatch stopwatch = Stopwatch.createStarted();
            T result = s3Call.apply(bucket, key);
            log.info("Retrieved {} from {}/{} in {}", outputDescription, bucket, key, logFriendlyDuration(stopwatch.elapsed()));
            return result;
        });
    }
}
```

As you can see, this client does many of the things we previously listed as advantages of service adapters:
- it adds helpful standardized logging and decorates AWS exceptions with more detailed exception messages
- it abstracts business code from the migration from AWS SDK v1 to v2 (many of these methods would be a bit more complicated in v1, especially the stream listing
  method because in the v1 SDK the client had fewer high-level features)
- it facilitates the use of some APIs by offering simpler shorthands

Image you have 10 services accessing S3 through this wrapper, now they will all benefit from this and you'll have the same logging and error handling everywhere. Truly worth the effort.

There is some amount of personalization into this client, but sometimes all you need is to add a few standard metrics, logging and error handling. One approach I have liked using in many of my services
is the `ServiceCall` pattern. Below is an example:

```java
/// Provides a consistent mechanism to decorate dependency calls with metrics and error handling.
@Log4j2
@RequiredArgsConstructor(access = AccessLevel.PRIVATE)
public class ServiceCall {
    private static final String TOTAL_FAILURES = "TotalFailures";
    private static final String PER_EXCEPTION_FAILURE_PREFIX = "Failure-";
    private static final String LATENCY = "Latency"; // single call latency
    private static final String END_TO_END_LATENCY = "EndToEndLatency"; // total latency including retries, if any

    public static ServiceCall of(String service, MetricPublisher metricPublisher) {
        return new ServiceCall(service, metricPublisher, DependencyException::new,
                RetryingCallableFactory.NO_RETRY, false, Ticker.systemTicker());
    }

    @NonNull private final String service;
    @NonNull private final MetricPublisher metricPublisher;

    @With
    @NonNull private final DependencyExceptionDecorator onException;

    @With(value = AccessLevel.PRIVATE)
    @NonNull private final RetryingCallableFactory retryingCallableFactory;

    @With(value = AccessLevel.PRIVATE)
    private final boolean shouldLogLatency;

    @VisibleForTesting
    @With(value = AccessLevel.PROTECTED)
    @NonNull private final Ticker ticker;

    public ServiceCall retryWith(Retry retry, CircuitBreaker circuitBreaker) {
        return withRetryingCallableFactory(new RetryingCallableFactory() {
            @Override
            public <T> Callable<T> wrap(Callable<T> callable, Metrics metrics) {
                return Decorators.ofCallable(new RetriedPublishingCallable<>(callable, metrics))
                        .withRetry(retry)
                        .withCircuitBreaker(circuitBreaker)
                        .decorate();
            }
        });
    }

    public ServiceCall enableLatencyLogging() {
        return withShouldLogLatency(true);
    }

    public void callVoid(String operation, Runnable runnable) {
        call(operation, () -> {
            runnable.run();
            return null;
        });
    }

    /// Calls the dependency and emits latency as well as success/failure metrics (both globally and exception-grained).
    public <T> T call(String operation, Callable<T> callable) {
        String fullOperationName = "%s.%s".formatted(service, operation);
        Stopwatch e2eStopwatch = Stopwatch.createStarted(ticker);

        return metricPublisher.getWithMetrics(fullOperationName, metrics -> {
            boolean success = false;
            try {
                TimedCallable<T> timedCallable = new TimedCallable<>(callable, metrics, ticker);
                T result = retryingCallableFactory.wrap(timedCallable, metrics).call();

                metrics.addCount(TOTAL_FAILURES, 0);
                success = true;

                return result;
            } catch (Exception e) {
                metrics.addCount(PER_EXCEPTION_FAILURE_PREFIX + e.getClass().getSimpleName(), 1);
                metrics.addCount(TOTAL_FAILURES, 1);
                throw onException.decorate(service, operation, e);
            } finally {
                Duration latency = e2eStopwatch.elapsed();
                metrics.addDuration(END_TO_END_LATENCY, latency);

                if (shouldLogLatency) {
                    log.info("Operation {} {} in {}", fullOperationName, success ? "succeeded" : "failed", humanReadableDuration(latency));
                }
            }
        });
    }

    /// Allows customizing the exception thrown by the dependency
    interface DependencyExceptionDecorator {
        DependencyException decorate(String service, String operation, Throwable cause);
    }

    private interface RetryingCallableFactory {
        RetryingCallableFactory NO_RETRY = new RetryingCallableFactory() {
            @Override
            public <T> Callable<T> wrap(Callable<T> callable, Metrics metrics) {
                return callable;
            }
        };

        <T> Callable<T> wrap(Callable<T> callable, Metrics metrics);
    }

    private record TimedCallable<T>(
            @NonNull Callable<T> underlyingCallable,
            @NonNull Metrics metrics,
            @NonNull Ticker ticker
    ) implements Callable<T> {
        @Override
        public T call() throws Exception {
            Stopwatch stopwatch = Stopwatch.createStarted(ticker);
            try {
                return underlyingCallable.call();
            } finally {
                metrics.addDuration(LATENCY, stopwatch.elapsed());
            }
        }
    }
}
```

Example usage:

```java
int maxAttempts = 3;
Retry retry = Retry.of("retry", RetryConfig.custom().maxAttempts(maxAttempts).build());
CircuitBreaker circuitBreaker = CircuitBreaker.ofDefaults("circuitBreaker");

ServiceCall.of("MyDependency", metricPublisher)
        .retryWith(retryConfig, circuitBreaker)
        .call("SomeApi", callable);
```

With such an adapter, I get the same standard features for all my dependencies which calls I wrapped using it:
- standard logs and metrics (success, failure, latency)
- a single way to configure retries. In some cases, your dependency won't have a client, only an HTTP endpoint and you have to do everything else, so having an easily reusable
  retry functionality on the adapter is good, but in case your dependency already has a good SDK, you may just use the SDK's built-in capabilities. For example for AWS, I always use native retries
  because it makes it easier to retry the right exceptions depending on their status code and retriability, as well as having²² good defaults for the retry config, adapted to each API.

If you have 15 dependencies, having such an adapter makes it easy to build a dashboard monitoring the basic health of all your dependencies because they all have a common set of metrics. It also makes investigations easier
because you know you can rely on a standard log pattern for all services. Whenever you improve or enrich this adapter, all dependencies using it can reap the benefits at once.

Now, there is a debate on whether this is the best implementation or not. Among alternatives, you could also:
- generate a `Proxy` at runtime. This would allow you using the same interface as the native client but with decorated behaviour. The proxy could allow plugging arbitrary listeners to publish metrics, logs etc.
- use annotations to decorate the behaviour of the native client. Similar as above but different implementation. In general it would work with AspectJ, or a compile-time annotation processor.
- use some library that gives you most of what you need

The reason why I have historically preferred my explicit `ServiceCall` approach is the following:
- yes, I have to own some code myself even though similar code exists many times over, however it's small and simple code, and owning it means I can add anything I want
  to it whenever I need to. I don't like depending on an external library for a small thing that I could do myself if it's likely that I'll need customizations that will be impossible or inconvenient
  to attain with a generic solution.
- I often don't like magic proxies. I find them more brittle and easy to break. If I wrap using a proxy, it's harder for me to test that the behaviour of the instance I will indeed use in production
  is what I want. On the flip side, if I want to test it with my explicit approach, I'll have to test it in every single one of my wrapped dependencies. If there is a strong way to guarantee that all the calls
  that I intend to be wrapped will indeed be wrapped, I'm fine using that.
- a proxy/annotation is not always enough on its own. You'll often need to write an explicit wrapper to get the advantages of services wrappers that aren't covered by `ServiceCall` such as encapsulating business logic,
  adding higher-level methods etc. Besides, again some APIs do not have a client, so you'll need to write one from scratch. Because of that, `ServiceCall` often naturally fits in the implementation of the wrapper client,
  whereas it would generally be more difficult (if possible at all) to use both a manually written adapter, and a proxy/annotation pattern.