# Smaller deployment options for GraalVM native image

*Estimated time: 15 minutes.*

Let's build a small microservice application responding to HTTP queries with the same business logic of generating prime numbers:

```
mn create-app primes-web
cd primes-web
```

Let's create a controller responsible for out business logic. Put the following class into the
`src/main/java/primes/web/PrimesController.java` file:


```
package primes.web;

import io.micronaut.http.annotation.Controller;
import io.micronaut.http.annotation.*;

import java.util.stream.*;


import java.util.*;

@Controller("/primes")
public class PrimesController {

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

    public static boolean isPrime(long n) {
    return LongStream.rangeClosed(2, (long) Math.sqrt(n))
            .allMatch(i -> n % i != 0);
    }

    public static List<Long> primeSequence(long min, long max) {
    return LongStream.range(min, max)
            .filter(PrimesController::isPrime)
            .boxed()
            .collect(Collectors.toList());
    }

}
```

You can build the app:
```
./gradlew build
```

Run it and open the app on port `8080` url: `/primes/random/100`:
```
java -jar build/libs/primes-web-0.1-all.jar
```

You can also build the native image of it:
```
./gradlew nativeImage
```
and run it with:

```
./build/native-image/application
```

Let's look at the application file:

```
ls -lah build/native-image/application
-rwxrwxr-x. 1 opc opc 74M Nov 25 01:11 build/native-image/application
```

We can compress the application with, for example, the upx utility: https://upx.github.io/

Run the upx with various options to check the speed/compression ratio, for example:
```
upx -k -8 build/native-image/application
                       Ultimate Packer for eXecutables
                          Copyright (C) 1996 - 2020
UPX 3.96        Markus Oberhumer, Laszlo Molnar & John Reiser   Jan 23rd 2020

        File size         Ratio      Format      Name
   --------------------   ------   -----------   -----------
  76758608 ->  21490044   28.00%   linux/amd64   application

Packed 1 file.
```

Now the application is smaller:
```
ls -lah build/native-image/application
-rwxrwxr-x. 1 opc opc 21M Nov 25 01:11 build/native-image/application
```

But it still starts well in a few ms:

```
build/native-image/application
01:19:06.684 [main] INFO  i.m.context.env.DefaultEnvironment - Established active environments: [oraclecloud, cloud]
01:19:06.699 [main] INFO  io.micronaut.runtime.Micronaut - Startup completed in 25ms. Server Running:
```


Next, we'll try to explore various other deployment options for native images.
