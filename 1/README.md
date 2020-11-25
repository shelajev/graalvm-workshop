# Building a simple GraalVM native image

*Estimated time: 15 minutes.*

GraalVM native image can process your application compiling it ahead of time into a standalone executable.

Some of the benefits you get from it are:
* small standalone distribution not requiring a JDK
* instant startup
* lower memory footprint

Let's illustrate this with a quick example.

We'll create a Micronaut application to compute sequences of prime numbers.

Make sure Micronaut is installed and GraalVM is installed and are of the correct versions.  

```
mn --version
Micronaut Version: 2.2.0
```

```
java --version
java 11.0.9 2020-10-20 LTS
Java(TM) SE Runtime Environment GraalVM EE 20.3.0 (build 11.0.9+7-LTS-jvmci-20.3-b06)
Java HotSpot(TM) 64-Bit Server VM GraalVM EE 20.3.0 (build 11.0.9+7-LTS-jvmci-20.3-b06, mixed mode, sharing)
```

```
native-image --version
GraalVM Version 20.3.0 EE (Java Version 11.0.9+7-LTS-jvmci-20.3-b06)
```

Create the application using Micronaut:
```
mn create-cli-app primes; cd primes
```

Create and edit the `src/main/java/primes/PrimesComputer.java` file:

```
package primes;
import javax.inject.Singleton;
import java.util.stream.*;
import java.util.*;
@Singleton
public class PrimesComputer {
    private Random r = new Random(41);
    public List<Long> random(int upperbound) {
        int to = 2 + r.nextInt(upperbound - 2);
        int from = 1 + r.nextInt(to - 1);
        return primeSequence(from, to);
    }
    public static List<Long> primeSequence(long min, long max) {
    return LongStream.range(min, max)
            .filter(PrimesComputer::isPrime)
            .boxed()
            .collect(Collectors.toList());
    }
    public static boolean isPrime(long n) {
        return LongStream.rangeClosed(2, (long) Math.sqrt(n))
                .allMatch(i -> n % i != 0);
    }
}
```

Edit the `src/main/java/primes/PrimesCommand.java` file:

```
package primes;
import io.micronaut.configuration.picocli.PicocliRunner;
import io.micronaut.context.ApplicationContext;
import picocli.CommandLine;
import picocli.CommandLine.Command;
import picocli.CommandLine.Option;
import picocli.CommandLine.Parameters;
import javax.inject.*;
import java.util.*;
@Command(name = "primes", description = "...",
        mixinStandardHelpOptions = true)
public class PrimesCommand implements Runnable {
    @Option(names = {"-n", "--n-iterations"}, description = "How many iterations to run")
    int n;

    @Option(names = {"-l", "--limit"}, description = "Upper limit for the sequence")
    int l;

    @Inject
    PrimesComputer primesComputer;

    public static void main(String[] args) throws Exception {
        PicocliRunner.run(PrimesCommand.class, args);
    }
    public void run() {
        for(int i =0; i < n; i++) {
            List<Long> result = primesComputer.random(l);
            System.out.println(result);
        }
    }
}
```

Remove the tests because we changed the functionality of the main command:
```
rm src/test/java/primes/PrimesCommandTest.java
```

Now we can build this Micronaut project to get the jar file with our functionality:
```
./gradlew build
```

Test the application that it prints the prime numbers:

```
java -jar build/libs/primes-0.1-all.jar -n 1 -l 100
[53, 59, 61, 67, 71, 73]
```

Now you can build the native image too:

```
./gradlew nativeImage
```

Micronaut includes a Gradle plugin to invoke the `native-image` utility and configure its execution.

You can find the resulting executable in `build/native-image/application`.

Inspect it with the `ldd` utility and check that it's linked to the OS libraries.

```
ldd build/native-image/application
```

Inspect its file type:

```
file build/native-image/application
```


If you have GNU time utility (`brew install gtime` on Macos), you can time the execution and the memory usage of the process. Compare the following:

```
/usr/bin/time -v java -jar build/libs/primes-0.1-all.jar -n 1 -l 100
```
vs.

```
/usr/bin/time -v build/native-image/application -n 1 -l 100
```

Next, we'll try to explore some more options how to configure the build process for native images.
