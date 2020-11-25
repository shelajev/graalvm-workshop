# For best peak performance use GraalVM with the JIT

*Estimated time: 12 minutes.*

Let's continue to experiment on the example project we created before.

Locate the sample project we created:

```
cd primes-web
```

If you haven't done it yet, build the app:
```
./gradlew build
```

And for this section we don't need the native images anymore.

--------------------------

Check that your `java` comes from a GraalVM distribution:
```
java -version
```

That should print something about GraalVM.

The application we've built is a jar file now and it's located at:
```
build/libs/primes-web-0.1-all.jar
```

We can run this application using GraalVM:

```
java -jar build/libs/primes-web-0.1-all.jar
```

Now this will run it with the JIT, so if we want to do any measurements for peak performance, we need to include the warmup into account.

Let's hit the application with the load for a few minutes so everything that needs compiling would hit the JIT.
```
hey -z 180s http://localhost:8080/primes/random/100
```

And let's do the measurement run too:
```
hey -z 60s http://localhost:8080/primes/random/100
```

Explore the output. Note that the numbers aren't comparable to the ones from before because we didn't limit the resources this time. If you have a powerful machine, this is as fast as it gets.


Now if you have another JDK available, you can repeat this test with it. We'll use one installed by sdkman (don't make it the default, it'll mess up the paths).

```
~/.sdkman/candidates/java/11.0.8-open/bin/java -jar build/libs/primes-web-0.1-all.jar
```

And the same test again, let's warm it up:
```
hey -z 180s http://localhost:8080/primes/random/100
```

And let's do the measurement run too:
```
hey -z 60s http://localhost:8080/primes/random/100
```

Explore the output.

Now these 2 measurements you can compare to figure out what runtime is better at optimizing the code of the application.


Next, we'll try to find out which optimizations GraalVM applied to our code and how to debug whether the code hits the JIT and so on.
