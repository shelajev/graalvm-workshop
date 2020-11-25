# GC options for GraalVM native image

*Estimated time: 15 minutes.*

In the previous part we've build a small microservice that can respond to the HTTP traffic.
Let's continue using it and try to explore the GC options GraalVM offers.

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

Let's run the application and apply some load to it so we can explore how it behaves under load.
We'll use the `hey` load generation tool (https://github.com/rakyll/hey).
There are binaries on the github page, or you can install with `go get hey` (if you have go).


You can run the application as is on the host machine, for example:
Run your app (use `&` at the end to make it run in the background or use 2 terminals/connections to run commands in parallel):
```
./build/native-image/application &
```

Run the load tool for 60s to get the measurements.
```
hey -z 60s http://localhost:8080/primes/random/100
```

However it makes sense to limit the available resources to simulate more cloud like deployment.

We'll use the docker image we've built before.

Run the following command to run the native image of our application:
```
docker run --rm -p 8080:8080 --memory="256m" --memory-swap="256m" --cpus=1 primes-web:slim
```

Note we're restricting the memory to 256m, disable the swap and limit it to have 1 CPU. It's a pretty constrained environment.

Now you can get the measurement, using the similar `hey` command:

```
hey -z 60s http://localhost:8080/primes/random/100
```

Edit the `build.gradle` file to include the following `nativeImage` section:
```
nativeImage {
  args("--gc=G1")
}
```
Run the build:
```
./gradlew nativeImage
```

We can use the same `Dockerfile.slim` to build the image (the app on the host has changed):
```
docker build -f Dockerfile.slim -t primes-web:g1gc .
```

Run this new docker image:
```
docker run --rm -p 8080:8080 --memory="256m" --memory-swap="256m" --cpus=1 primes-web:g1gc
```

And apply the load as before:  
```
hey -z 60s http://localhost:8080/primes/random/100
```

Explore the output.


\* Add the printGC / verboseGC flags we looked at in the previous chapter to the g1gc based image to look at the heap configuration. 

Next, we'll try to explore how to improve performance of the native images.
