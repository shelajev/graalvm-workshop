# Deployment options for GraalVM native images

*Estimated time: 15 minutes.*

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

Build the native image of it again (let's not use the upxed version, if you used the `-k` option to keep the backup, the executable should be in `build/native-image/application.~`).
```
./gradlew nativeImage
```
and locate the binary of the app (if needed move the upx backup file: `mv build/native-image/application.~ build/native-image/application`):

```
./build/native-image/application
```

Explore the output of `ldd` to check which libraries it's linked against:
```
ldd ./build/native-image/application
```

Let's package it into a docker image. One of the benefits of the native image is that it offers smaller deployment sizes, so we'll use a `slim` image.

Create a `Dockerfile.slim` file with the following:

```
FROM oraclelinux:8-slim
COPY build/native-image/application app
ENTRYPOINT ["/app"]
```

Build the image:
```
docker build -f Dockerfile.slim -t primes-web:slim .
```

Install the `dive` utility to inspect the images: https://github.com/wagoodman/dive#installation

Run it on the new image and explore the output:
```
dive primes-web:slim
```

The image doesn't have much except our application. We can still run it and access the app on port 8080.
```
docker run --rm -p 8080:8080 primes-web:slim
```

However we can do even better. One way is to link all necessary libraries statically, except `libc`. Which we'll depend on from the environment.

Edit the `build.gradle` file to configure the native-image use through the Micronaut's Gradle plugin. Add the following section:

```
nativeImage {
  args("-H:+StaticExecutableWithDynamicLibC")
}
```
Build it again and explore the output of `ldd`:
```
./gradlew nativeImage
```

```
ldd ./build/native-image/application
```

Now we can package it into an even smaller docker image, let's call this `Dockerfile.distroless`

```
FROM gcr.io/distroless/base
COPY build/native-image/application app
ENTRYPOINT ["/app"]
```

Build the image and explore it with `dive`:

```
docker build -f Dockerfile.distroless -t primes-web:distroless .
```

Look at the image efficiency score!
```
dive primes-web:distroless
```

Running it is just as simple:
```
docker run --rm -p 8080:8080 primes-web:distroless
```

Use the app a few times, explore the docker container stats in `docker stats` or some monitoring service like `cadvisor`.


\* A bonus point exercise is to build a statically linked native image which is completely self-contained and can be used in the empty docker image: `FROM scratch`. This is possible on Linux only, and requires some prerequisites described in [the docs](https://www.graalvm.org/reference-manual/native-image/StaticImages/).

But in a nutshell you can pass `--static --libc=musl` and get a statically linked binary.

Please take your time to explore it.

Next, we'll try to explore various options for configuring the runtime for native images.
