# Class initialization strategy for GraalVM native image

*Estimated time: 15 minutes.*

One of the most misunderstood features of native image is the class initialization strategy.
In this part of the workshop we'll try to explain it a little bit.

Classes of your application need to be initialized before being used. The lifecycle of the native image is split into two parts: build time and run time.

By default classes are initialized at runtime. Sometimes it makes sense for optimization or some other reasons to initialize some classes at runtime.

Let's explore an example application consisting of a few classes to better understand the implications of init at runtime or build time and how to configure the init strategy.

Here's our program:

```
import java.nio.charset.*;


public class Main {

  public static void main(String[] args) {
    A.b.doit();

  }
}


class A {

  public static B b = new B();

}

class B {

  private static final Charset UTF_32_LE = Charset.forName("UTF-32LE");

  public void doit() {
    System.out.println(UTF_32_LE);
  }
}
```

It consists of 3 classes: Main, calling `A.b.doit()`; A, holding a reference to a B instance in a static field; and B, holding a reference to a Charset - UTF-32LE.

Run it and explore the output:

```
javac Main.java
java Main
```

Build a native image of it, run it, and explore the output:

```
native-image -cp . Main
./main
```

It breaks with the following exception, because the Charset "UTF_32_LE" is not by default included in the native image and is not found:

```
Exception in thread "main" java.lang.ExceptionInInitializerError
	at com.oracle.svm.core.classinitialization.ClassInitializationInfo.initialize(ClassInitializationInfo.java:291)
	at A.<clinit>(Main.java:15)
	at com.oracle.svm.core.classinitialization.ClassInitializationInfo.invokeClassInitializer(ClassInitializationInfo.java:351)
	at com.oracle.svm.core.classinitialization.ClassInitializationInfo.initialize(ClassInitializationInfo.java:271)
	at Main.main(Main.java:7)
Caused by: java.nio.charset.UnsupportedCharsetException: UTF-32LE
	at java.nio.charset.Charset.forName(Charset.java:529)
	at B.<clinit>(Main.java:21)
	at com.oracle.svm.core.classinitialization.ClassInitializationInfo.invokeClassInitializer(ClassInitializationInfo.java:351)
	at com.oracle.svm.core.classinitialization.ClassInitializationInfo.initialize(ClassInitializationInfo.java:271)
	... 4 more
```

One way to resolve this issue is to include all charsets like the following does, try it:
```
native-image -H:+AddAllCharsets -cp . Main
```

However we're more interested in the class init details right now. Use the `-H:+PrintClassInitialization` to check how are classes initialized:

```
native-image -H:+PrintClassInitialization -cp . Main
```

Check the output:
```
cat reports/run_time_classes_*
```

You can for example see classes like:
```
com.oracle.svm.core.containers.cgroupv1.CgroupV1Subsystem
com.oracle.svm.core.containers.cgroupv2.CgroupV2Subsystem
```
which are used for determining the V1/V2 cgroup resources availability when running in containers.
And also our classes `A` and `B`.

Initializing the class means running it's `<clinit>` so it tries to load the charset and breaks at runtime.

What we can do in this case is move the initialization to build time, which will succeed because build time is a Java process and it'll load the chatset without any problems.

```
native-image --initialize-at-build-time=A,B -cp . Main
```

The classes are initialized at build time, the Chatset instance is written out to the image heap and can be used at runtime.
Sometimes objects instantiated during the build class initialization cannot be written out and used at runtime:
* opened files
* running threads
* opened network sockets
* Random instances

If the analysis sees them in the image heap -- it'll notify you and ask to initalize the classes holding them at runtime.

For example, if we modify the code to be:
```
import java.nio.charset.*;


public class Main {

  public static void main(String[] args) {
    A.b.doit();
    System.out.println(A.t);
  }
}


class A {

  public static B b = new B();

  public static Thread t;

  static {
    t = new Thread(()-> {
      try {
        Thread.sleep(30_000);
      } catch (Exception e){}
    });
    t.start();
  }

}

class B {

  private static final Charset UTF_32_LE = Charset.forName("UTF-32LE");

  public void doit() {
    System.out.println(UTF_32_LE);
  }
}
```

Building the native image like before will fail, please observe how:
```
native-image --no-fallback --initialize-at-build-time=A,B -cp . Main
```

Balancing initialization can be a bit tricky, so by default GraalVM initializes classes at runtime. So for this example it's good to have initialize only `B` at build time.

```
native-image --no-fallback --initialize-at-build-time=B -cp . Main
```

Running it will take 30 seconds now because of the `Thread.sleep`

```
./main
UTF-32LE
Thread[Thread-0,5,main]
~/init-strategy
```


Next, we'll try to explore some more options how to configure the class initialization strategy for native images.
