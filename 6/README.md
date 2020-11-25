# Configuring memory used by GraalVM native images

*Estimated time: 5 minutes.*

In the previous part we've build a small microservice that can respond to the HTTP traffic.
Let's continue using it and try to explore some options how we can deploy it differently.

Locate the sample project we created:

```
cd primes-web
```

If you haven't done it yet, build the app:
```
./gradlew build
```

If necessary, build the native image of it again:
```
./gradlew nativeImage
```
and locate the binary of the app at `./build/native-image/application`.

The native executable uses managed memory just as your application would expect. It means that the allocation/GC of the objects are handled by a GC implementation.

You can configure the heap configuration options similar to running Java application on HotSpot.

Use the `-Xmx` and `-Xmn` options to configure the heap size and the young generation size for your application.

Explore various options to see how low can you go without crashing? or with still reasonably decent startup time (<100ms)?

For example run with the max heap size of 32M:
```
./build/native-image/application -Xmx32m
```

The default values for these options are: 8M for the young generation, unlimited (limited by the hw resources) for the max heap size. You can control it with the `-XX:MaximumHeapSizePercent` option.

Enable the logging for GC using the following flags:

```
-XX:+PrintGC - print basic information for every garbage collection
-XX:+VerboseGC - can be added to print further garbage collection details
```

Try using the application for a bit to trigger the GC, observe the logs trying to make sense of them.

```
./build/native-image/application -XX:+PrintGC -XX:+VerboseGC -Xmx32m
```

Next, we'll try to explore various options for configuring the runtime for native images.
