# Profile guided optimizations for GraalVM native image

*Estimated time: 15 minutes.*

Let's continue to experiment on the example project we created before.

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

--------------------------

One very interesting feature of the GraalVM native images is the ability to use profile guided optimizations to improve the performance of the native executables produced with native image.

In a similar fashion to the JIT compiler gathering the profile, we can build an instrumented image, apply the desired load to it (your load tests, benchmark suite, or a slice of real workload), collect the profile, and use it to build a more optimized executable for the particular workloads.

Here's how you do it:

We need to supply the `--pgo-instrument` option to the native image build, then we build native image using the `--pgo` option.

For simplicity and consistency we'll edit the `build.gradle` manually as before, but the separated nature of the build (2 stages) nicely maps into the separate jobs in CI or scripting.

Edit the `build.gradle` to include the following:

```
nativeImage {
  args("--pgo-instrument")
  args("--gc=G1")
}
```

Build the image:
```
./gradlew nativeImage
```

Now run it and apply the load:

```
build/native-image/application
```
The load can be shorter because we just need the profile:
```
hey -z 30s http://localhost:8080/primes/random/100
```

Stop the application with `Ctrl+C`, look at the profile file:

```
ls -la default.iprof
```

We'll use it now to build the final image with the profile guided optimizations:

Edit the `build.gradle` (note you can have several profile files comma separated there):

Also note the `../..` it's because the building happens in the `build/native-image` directory.
```
nativeImage {
  args("--pgo=../../default.iprof")
  args("--gc=G1")
}
```
Build the image:
```
./gradlew nativeImage
```

Test it works:
```
./build/native-image/application
```

Package it into the docker image.
We can use the same `Dockerfile.slim` to build the image (the app on the host has changed):
```
docker build -f Dockerfile.slim -t primes-web:pgo .
```

Run this new docker image:
```
docker run --rm -p 8080:8080 --memory="256m" --memory-swap="256m" --cpus=1 primes-web:pgo
```

And apply the load as before:  
```
hey -z 60s http://localhost:8080/primes/random/100
```

Explore the output.


Next, we'll try to explore how to improve peak performance of application even further.
