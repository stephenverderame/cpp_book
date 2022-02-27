# Valgrind

Valgrind is a suite of profiling and performance tools that's more recent and more powerful than gprof. Besides
being able to effectively profile multithreaded code, valgrind also has tools for finding memory leaks and unitialized 
data, inconsistant lock ordering and other threading issues, and much more.

I'll show a few examples, but I'll refer you to the [Valgrind User Manual](https://valgrind.org/docs/manual/manual.html) for more
information. The manual is pretty clear, and I haven't really used this enough to discuss much in detail.

To use it, we first have to compile whatever program we want to analyze with debugging symbols. 
Technically,  you don't *need* debugging symbols, but this just makes the output comprehensible.
For GCC, this would be compiling with the `-g` flag.
Unlike gprof, we don't need any specific compilation option or linking option.
Next, we can simply run the program by passing it to an argument of Valgrind.

The general command structure looks like this:
```
valgrind --tool=<tool-name> [--<tool flags>] <program name> [program args]
```

When running a program with valgrind, you can expect the program to be **significantly** slower. Valgrind gives an estimate of `20x` to `30x` slower, but I wouldn't sweat it if it takes longer. Some tools are faster or slower than others.
It's also important to remember that Valgrind (or any analysis tool for that matter) cannot be perfect.
(Halting Problem, Rice's Theorem, bugs in Valgrind itself, etc.)

Valgrind will run our program with some extra information displayed on the console depending on the tool used.
For each line Valgrind displays, you will notice a `==<number>==` at the start of each line.
This number is the process id.

It may be helpful to save logs to a file, which can be done using the `--log-file=<filename>` flag. By default,
Valgrind logs its information to `stderr`, so the redirect both `stdout` and `stderr` to a file you can use the following:

```
<command> > allout.txt 2>&1
```

## Memchek

Memcheck is the default tool for Valgrind, to when running memcheck you don't need the `--tool` flag.
Memcheak looks for invalid memory usage (reading/writing to invalid data, memory leaks, usage of unitialized data etc.)

If we run the program using the following command:

```
valgrind --leak-check=full <program>
```

When Memcheck is finished, it will summarize any memory leaks it has found. The term "lost"
essentially means that memory is leaked. More specificly "indirectly lost" means that memory
is lost as a result of another memory leak (such as children nodes of a tree), "still reachable"
characterizes memory that is still accessible by globals or static variables when the program
exits. "Possibly lost" should be treated as a "definetely lost" is most cases. The distinction
is made because Valgrind can't tell if it's a memory leak or if you are doing something
clever with pointers (in most cases).

The `--leak-check=full` parameter will tell Valgrind to display detailed information for any memory
leaks it finds instead of just summarizing how many it finds.

## Callgrind

Callgrind is a callgraph-based profiling tool. We can run it like so 

```
valgrind --tool=callgrind <program>
```

Callgrind will log its output to a file named `callgrind.out.<pid>` unless this is changed using the `--out-file` flag.

To view this data, you can use `kcachegrind` (`qcachegrind` on Windows) to display the callgraph and flat profile in a graphical format.

## Helgrind and DRD

Helgrind and DRD are both tools for thread error detection. They may detect different types of issues, so it may be worth it to use
both of these. The errors these tools detect are misuse of the `pthreads` API, data races, and potential deadlock. For us C++ programmers, we don't have to worry too about the first class of errors too much since most of `pthreads` will be abstracted from us by
the standard library or boost.

Similar to callgrind, you can use these with the flags `--tool=helgrind` and `--tool=drd`. Both of these tools work on implementations
that use `pthreads`.

Now it should be noted that Helgrind can generate a lot of false positives. This mostly resolves around the fact that Helgrind
doesn't seem to play nice non-pthread synchronization primitives such as atomics, or pthread condition variables. So if you use this, I might suggest to focus more on any issues it finds with locking order.

DRD seems to be slower than Helgrind, but supports more synchronization primitives.

# Sanitizers

Sanitizers work similarly to Valgrind, but often are more efficient due to the face that unlike valgrind, they are not a purely runtime
tool. Depending on the tool, they claim to have a roughly `2x` slowdown.

These tools come with Clang and GCC, and work by simply adding compilation **and link** flags to your program,
and then running the program normally.

When using the sanitizers, they reccomend using optimization level 1 (flag `-O1`, that's an "oh" not a 0) which will disable
some optimizations (such as inlining) that can make analysis more difficult.

Some sanitizers (such as Asan (address sanitizer), UBsan (undefined behavior sanitizer), and Lsan (leak sanitizer)) can be run together
while others like Tsan (thread sanitizer) need separable builds.

## Address and Memory Sanitizer

Address sanitizer is similar to Valgrind's Memcheck. It works by applying the flag `-fsantizie=address`. For C++,
they reccommend also using the argument `-fno-omit-frame-pointer`. It may also be helpful to disable
tail call elimination with `-fno-optimize-sibling-call`.

Memory sanitizer checks for unitialized reads. Use `-fsanitize=memory`.

## Thread Sanitizer

This is similar to Valgrind's Helgrind and DRD. We enable this one by applying the flag `-fsanitize=thread`.
Thread Sanitizer checks for data races, and I'd reccomend this one over Valgrind if that's what you're
interested in.

## Undefined Behavior Sanitizer

As the name implies, this one checks for undefined behavior. We can compile with the `-fsanitize=undefined` flag.
You might also want to use `-fsanitize=nullability` in conjunction.

## Leak Sanitizer

As the name implies, this one checks for memory leaks. To use this one, you guessed it, link with `-fsanitize=leak`.

---
For more information on Sanitizers, check out [clang's compiler documentation](https://clang.llvm.org/docs/index.html) and for more information on Cachegrind, check out [their website](https://valgrind.org/).