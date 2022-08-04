# Periodic tasks post-Loom

old

```java
var scheduler = Executors.newScheduledThreadPool(1);
var future = scheduler.scheduleAtFixedRate(() -> {
    System.out.println("Hello, World!");
}, 1, 2, TimeUnit.SECONDS);
```

new

```java
var scope = new StructuredTaskScope<>();
var future = scope.fork(() -> {
    for (var deadline = Instant.now().plusSeconds(1);;
             deadline = deadline.plusSeconds(2)
    ) {
        Thread.sleep(Duration.between(Instant.now(), deadline));
        System.out.println("Hello, World!");
    }
    return null; // Unreachable
});
```

But... that's so loooong...

Abstraction to the rescue!

```java
public static Callable<?> repeatAtFixedRate(
    Runnable command,
    Duration initialDelay,
    Duration period
) {
    for (var deadline = Instant.now().plus(initialDelay);;
             deadline = deadline.plus(period)
    ) {
        Thread.sleep(Duration.between(Instant.now(), deadline));
        command.run();
    }
    return null; // Unreachable
}

...

var scope = new StructuredTaskScope<>();
var future = scope.fork(repeatAtFixedRate(() -> {
    System.out.println("Hello, World!");
}, Duration.ofSeconds(1), Duration.ofSeconds(2)));
```

Of course the same goes for scheduleWithFixedDelay

```java
public static Callable<?> repeatWithFixedDelay(
    Runnable command,
    Duration initialDelay,
    Duration delay
) {
    Thread.sleep(initialDelay);
    for (;;) {
        command.run();
        Thread.sleep(delay);
    }
    return null; // Unreachable
}

...

var scope = new StructuredTaskScope<>();
var future = scope.fork(repeatWithFixedDelay(() -> {
    System.out.println("Hello, World!");
}, Duration.ofSeconds(1), Duration.ofSeconds(2)));
```