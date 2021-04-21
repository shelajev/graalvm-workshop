# Configuring JDK proxies for Native Image with the agent

*Estimated time: 10 minutes.*

One of the language features that require explicit configuration besides Reflection is JDK proxies. 
In the same way we configure usages of Reflection API with the javaagent we can register the classes which will be proxied at runtime. 

Imagine you have a class like this in the `Main.java`:

```
import java.lang.reflect.Proxy;
import java.util.Map;

public class Main {
  public static void main(String[] args) throws Exception {
    String className = args[0];
    Map proxyInstance = (Map) Proxy.newProxyInstance(
      Main.class.getClassLoader(),
      new Class[] {
        Class.forName(className)
      },
      (proxy, method, methodArgs) -> {
        if (method.getName().equals("get")) {
          return 42;
        } else {
          throw new UnsupportedOperationException(
            "Unsupported method: " + method.getName());
        }
      });

    int fortytwo = (int) proxyInstance.get("hello"); // 42
    System.out.println("Got " + fortytwo);
    try {

      proxyInstance.put("hello", "world"); // exception
    } catch (Exception e) {
      System.out.println("Expected an exception, got an exception");
      return;
    }
    throw new RuntimeException("Expected an exception, didn't get one");

  }
}
```


The main method creates a JDK proxy for the class the name of which is passed via the arguments. Note if we use the class literals static analysis is smart enough to figure it out and include necessary classes itself.

We can run the example
```
javac Main.java
java Main java.util.Map

```
As expected the program produces "Got 42" and then receives the exception and logs it. 

Let's produce the config using the assisted configuration agent: 

```
mkdir -p META-INF/native-image
java -agentlib:native-image-agent=config-output-dir=META-INF/native-image Main java.util.Map
Got 42
Expected an exception, got an exception
```

The `META-INF/native-image` directory now has the configuration files:

```
cat META-INF/native-image/reflect-config.json
[
{
    "name":"java.util.Map"
}
]

cat META-INF/native-image/proxy-config.json
[
    ["java.util.Map"]
]

```

The configuration for the proxies is incredibly straightforward, but the best thing is that you can generate it by running the application with the javaagent. 

Native image build succeeds now: 
```
native-image Main
[main:1651662]    classlist:   1,436.07 ms,  0.96 GB
[main:1651662]        (cap):     732.29 ms,  0.96 GB
[main:1651662]        setup:   3,215.70 ms,  0.96 GB
[main:1651662]     (clinit):     208.68 ms,  1.22 GB
[main:1651662]   (typeflow):   5,217.46 ms,  1.22 GB
[main:1651662]    (objects):   4,049.75 ms,  1.22 GB
[main:1651662]   (features):     275.95 ms,  1.22 GB
[main:1651662]     analysis:   9,984.87 ms,  1.22 GB
[main:1651662]     universe:     533.26 ms,  1.26 GB
[main:1651662]      (parse):     946.22 ms,  1.26 GB
[main:1651662]     (inline):   1,330.67 ms,  1.26 GB
[main:1651662]    (compile):  12,259.91 ms,  4.02 GB
[main:1651662]      compile:  15,487.70 ms,  4.02 GB
[main:1651662]        image:   1,316.30 ms,  4.02 GB
[main:1651662]        write:     288.94 ms,  4.02 GB
[main:1651662]      [total]:  32,475.62 ms,  4.02 GB
```

And running the code works as well: 

```
./main java.util.Map
Got 42
Expected an exception, got an exception
```

Next, we'll try to explore some more options how to configure the class initialization strategy for native images.
