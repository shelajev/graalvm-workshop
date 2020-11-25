# Assisted configuration for GraalVM native image

*Estimated time: 10 minutes.*

GraalVM native image build uses the closed universe assumption, which means all the bytecode in the application needs to be observed and analysed at the build time.

One area the analysis process is responsible for is to determine which classes, methods and fields need to be included in the executable. The analysis is static, so it might need some configuration to correctly include the parts of the program that use dynamic features of the language.

For example, classes and methods accessed through the Reflection API need to be configured. There are a few ways how these can be configured, but the most convenient way is the assisted configuration javaagent.

Imagine you have a class like this in the `ReflectionExample.java`:

```
import java.lang.reflect.Method;

class StringReverser {
    static String reverse(String input) {
        return new StringBuilder(input).reverse().toString();
    }
}

class StringCapitalizer {
    static String capitalize(String input) {
        return input.toUpperCase();
    }
}

public class ReflectionExample {
    public static void main(String[] args) throws ReflectiveOperationException {
        String className = args[0];
        String methodName = args[1];
        String input = args[2];

        Class<?> clazz = Class.forName(className);
        Method method = clazz.getDeclaredMethod(methodName, String.class);
        Object result = method.invoke(null, input);
        System.out.println(result);
    }
}
```


The main method invokes all methods whose names are passed as command line arguments.
Run it normally and explore the output.
```
java ReflectionExample StringReverser reverse "hello"
```

As expected, the method `foo` was found via reflection, but the non-existent method `xyz` was not found.

Let's build a native image out of it:

```
native-image --no-fallback ReflectionExample
```

*If you're interested ask the workshop leaders about the `--no-fallback` option*

Run the result and explore the output:

```
./reflectionexample StringReverser reverse "hello"
```

Writing a complete reflection configuration file from scratch is possible, but tedious.
Therefore, we provide an agent for the Java HotSpot VM that produces a reflection configuration file by tracing all reflective lookup operations.
It will also record all usages of the Java Native Interface (JNI), Java Reflection, Dynamic Proxy objects (`java.lang.reflect.Proxy`), or class path resources (`Class.getResource`).

You can specify the agent like the following:

create the directory for the configuration:
```
mkdir -p META-INF/native-image
```

Run the application:
```
java -agentlib:native-image-agent=config-output-dir=META-INF/native-image ReflectionExample StringReverser reverse "hello"
```

Explore the created configuration:

```
cat META-INF/native-image/reflect-config.json
```

You can also have multiple runs recorded and the config merged with specifying the `native-image-agent=config-merge-dir` option:

```
java -agentlib:native-image-agent=config-merge-dir=META-INF/native-image ReflectionExample StringCapitalizer capitalize "hello"
```

Building the native image now will make use of the provided configuration and configure the reflection for it.

```
native-image --no-fallback ReflectionExample
```

Run it and observe the output:
```
reflectionexample StringReverser reverse "joker"
```

This is a very convenient way to configure reflection and resources used by the application for building native images.

Next, we'll try to explore some more options how to configure the class initialization strategy for native images.
