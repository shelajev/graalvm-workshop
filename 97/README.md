# Exploring the polyglot runtime of GraalVM

*Estimated time: 10 minutes.*

Let's add JavaScript to the example project we created before.

Copy the project to have a clean state:
```
cp -R primes-web primes-js
cd primes-js
```

Edit the `src/main/java/primes/web/PrimesController.java` file to look like the following.

```
package primes.web;

import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.*;

import java.io.*;
import java.util.*;
import java.util.stream.*;

import org.graalvm.polyglot.*;

@Controller("/primes")
public class PrimesController {

    private final Engine sharedEngine = Engine.newBuilder().build();

    private ThreadLocal<Value> isPrimes = ThreadLocal.withInitial(() -> {

        Map<String, String> options = new HashMap<>();
        options.put("js.commonjs-require", "true");
        options.put("js.commonjs-require-cwd", new File(".").getAbsolutePath());


        Context cx = Context.newBuilder("js")
                            .allowAllAccess(true)
                            .allowHostAccess(HostAccess.ALL)
                            .allowPolyglotAccess(PolyglotAccess.ALL)
                            .allowExperimentalOptions(true)
                            .options(options)
                            .engine(sharedEngine)
                            .build();

        return cx.eval("js",
          "(function isPrime(num) {\n" +
          "   for(var i = 2; i < num; i++)\n" +
          "     if(num % i === 0) return false;\n" +
          "   return num > 1;\n" +
          "})"
        );
    });

    private Random r = new Random(41);

    @Get("/random/{upperbound}")
    public List<Long> random(int upperbound) {
        int to = 2 + r.nextInt(upperbound - 2);
        int from = 1 + r.nextInt(to - 1);

        return primeSequence(from, to);
    }

    @Get("/{from}/{to}")
    public List<Long> primes(@PathVariable int from, @PathVariable int to) {
        return primeSequence(from, to);
    }

    public boolean isPrime(long n) {
      return isPrimes.get().execute(n).asBoolean();
    }

    public List<Long> primeSequence(long min, long max) {
    return LongStream.range(min, max)
            .filter(this::isPrime)
            .boxed()
            .collect(Collectors.toList());
    }

}
```

This code defines a GraalVM Engine and a polyglot Context per thread in our Micronaut application.
Then we evaluate a snippet of JavaScript code, given as a String for simplicity, that contains the definition of the `isPrime` function.

The Java method can consume it then as if it was a normal Java object. We could even convert it to a `java.util.Function` function.

We have the project built normally with `./gradlew build` and can run it with the normally like this:

```
java -jar build/libs/primes-web-0.1-all.jar
```
