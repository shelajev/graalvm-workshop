# Exploring the GraalVM JIT

*Estimated time: 10 minutes.*

Let's continue to experiment on the example project we created before.

Locate the sample project we created:

```
cd primes-web
```

We have the project built and can run it with the `java` like this:

```
java -jar build/libs/primes-web-0.1-all.jar
```

--------------------------

There's a number of options available to turn on the additional logging about the compiler events, which you can find in the [docs](https://github.com/oracle/graal/blob/master/compiler/docs/Debugging.md).

One particular options is interesting to check if a certain piece of code you're interested in get to the JIT:
`-Dgraal.PrintCompilation=true`.

Run the application like this and we'll see a bunch of output that we can interpret:

```
java -Dgraal.PrintCompilation=true -jar build/libs/primes-web-0.1-all.jar
```

Now to inspect the actual optimizations the compiler does you can enable GraalVM to dump the compiler graphs for the code it processes.

```
java -Dgraal.Dump=:2 -Dgraal.MethodFilter=PrimesController.* -jar build/libs/primes-web-0.1-all.jar
```

This enables the dump and filters it to only contain the interesting classes in our app.

Apply the load:
```
hey -z 60s http://localhost:8080/primes/random/100
```
Check the log for the location of the files, a line like:

```
Dumping IGV graphs in /home/opc/primes-web/graal_dumps/2020.11.25.12.42.21.147
```
Download the `.bgv` files.

Run the `idealgraphvizualizer` locally, it's GUI application, load the graphs into it.

Explore the transformations.

Next, we'll try to learn about the polyglot nature of GraalVM.
