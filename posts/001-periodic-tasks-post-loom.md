# Periodic tasks post-Loom

old: minimize blocking - use ScheduledExecutorService

``` java
var scheduler = Executors.newScheduledThreadPool(1);
var future = scheduler.scheduleAtFixedRate(() -> {
    System.out.println("Hello, World!");
}, 1, 2, TimeUnit.SECONDS);
```

new: embrace blocking - use Thread.sleep()

``` java
try (var scope = new StructuredTaskScope<>()) {
    var future = scope.fork(() -> {
        for (var deadline = Instant.now().plusSeconds(1);;
                 deadline = deadline.plusSeconds(2)
        ) {
            Thread.sleep(Duration.between(Instant.now(), deadline));
            System.out.println("Hello, World!");
        }
        return null; // Unreachable
    });
}
```

Now use abstraction:

``` java
public static Callable<?> repeatAtFixedRate(
    Runnable command,
    Duration initialDelay,
    Duration period
) {
    return () -> {
        for (var deadline = Instant.now().plus(initialDelay);;
                 deadline = deadline.plus(period)
        ) {
            Thread.sleep(Duration.between(Instant.now(), deadline));
            command.run();
        }
        return null; // Unreachable
    };
}

...

// Usage:
try (var scope = new StructuredTaskScope<>()) {
    var future = scope.fork(repeatAtFixedRate(() -> {
        System.out.println("Hello, World!");
    }, Duration.ofSeconds(1), Duration.ofSeconds(2)));
}
```

Same goes for scheduleWithFixedDelay:

``` java
public static Callable<?> repeatWithFixedDelay(
    Runnable command,
    Duration initialDelay,
    Duration delay
) {
    return () -> {
        Thread.sleep(initialDelay);
        for (;;) {
            command.run();
            Thread.sleep(delay);
        }
        return null; // Unreachable
    };
}

...

// Usage:
try (var scope = new StructuredTaskScope<>()) {
    var future = scope.fork(repeatWithFixedDelay(() -> {
        System.out.println("Hello, World!");
    }, Duration.ofSeconds(1), Duration.ofSeconds(2)));
}
```