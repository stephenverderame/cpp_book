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

### Memcheck

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

The output of Memcheck is pretty self explanatory, and when using `--leak-check=full`, will display a stack trace for where
the memory leak occurs. By default, Memcheck can only tell us if uninitialized memory is used,
and not the cause of it. The argument `--track-origins=yes` enables tracing unitialized memory, but may slow down the runtime.

### Callgrind

Callgrind is a callgraph-based profiling tool. We can run it like so 

```
valgrind --tool=callgrind <program>
```

Callgrind will log its output to a file named `callgrind.out.<pid>` unless this is changed using the `--out-file` flag.

To view this data, you can use `kcachegrind` (`qcachegrind` on Windows) to display the callgraph and flat profile in a graphical format.
These tools provide you with GUIs that make looking at the data pretty easy. They do however only show the top 50 functions in a call graph.
If you need to see everything, `gprof2dot` works for the Callgrind output as well.

### Helgrind and DRD

Helgrind and DRD are both tools for thread error detection. They may detect different types of issues, so it may be worth it to use
both of these. The errors these tools detect are misuse of the `pthreads` API, data races, and potential deadlock. For us C++ programmers, we don't have to worry too about the first class of errors too much since most of `pthreads` will be abstracted from us by
the standard library or boost.

Similar to callgrind, you can use these with the flags `--tool=helgrind` and `--tool=drd`. Both of these tools work on implementations
that use `pthreads`.

Now it should be noted that Helgrind can generate a lot of false positives. This mostly resolves around the fact that Helgrind
doesn't seem to play nice non-pthread synchronization primitives such as atomics, or pthread condition variables. So if you use this, I might suggest to focus more on any issues it finds with locking order.

DRD seems to be slower than Helgrind, but supports more synchronization primitives.

Let's look at some example output from these tools. This one is a nice example from the [Valgrind DRD User Manual](https://valgrind.org/docs/manual/drd-manual.html)

```
...
==9466== Thread 3:
==9466== Conflicting load by thread 3 at 0x006020b8 size 4
==9466==    at 0x400B6C: thread_func (rwlock_race.c:29)
==9466==    by 0x4C291DF: vg_thread_wrapper (drd_pthread_intercepts.c:186)
==9466==    by 0x4E3403F: start_thread (in /lib64/libpthread-2.8.so)
==9466==    by 0x53250CC: clone (in /lib64/libc-2.8.so)
==9466== Location 0x6020b8 is 0 bytes inside local var "s_racy"
==9466== declared at rwlock_race.c:18, in frame #0 of thread 3
==9466== Other segment start (thread 2)
==9466==    at 0x4C2847D: pthread_rwlock_rdlock* (drd_pthread_intercepts.c:813)
==9466==    by 0x400B6B: thread_func (rwlock_race.c:28)
==9466==    by 0x4C291DF: vg_thread_wrapper (drd_pthread_intercepts.c:186)
==9466==    by 0x4E3403F: start_thread (in /lib64/libpthread-2.8.so)
==9466==    by 0x53250CC: clone (in /lib64/libc-2.8.so)
==9466== Other segment end (thread 2)
==9466==    at 0x4C28B54: pthread_rwlock_unlock* (drd_pthread_intercepts.c:912)
==9466==    by 0x400B84: thread_func (rwlock_race.c:30)
==9466==    by 0x4C291DF: vg_thread_wrapper (drd_pthread_intercepts.c:186)
==9466==    by 0x4E3403F: start_thread (in /lib64/libpthread-2.8.so)
==9466==    by 0x53250CC: clone (in /lib64/libc-2.8.so)
...
```

What we're looking at here is the report of a race condition. Race conditions require at least two accesses to the same data,
the first access displayed is shown with a precise stacktrace, the subsequent ones are shown with approximate stack traces
in between "segments". A segment in this sense is a sequence of reads and writes. 

To read the approximate stacktraces, it helps to read the stack from bottom to top, and start searching where the stack traces begin
to differ. In this example, that would point us to `thread_func` in `rwlock_race.c` from lines `28` to `30`.

DRD numbers each thread; these numbers are, as far as I can tell, somewhat arbitrary and really only useful for differentiating
threads in the report.

A useful flag on DRD is the `--exlusive-threshold=<time>` and `--shared-threshold=<time>` arguments. These allow you to set threshold durations,
in milliseconds, for exclusive/unique/writer locks and shared/reader locks, respectively. If a lock is held for longer than these
set thresholds, DRD will display an error telling you where the lock was taken and where it was released.

# Sanitizers

Sanitizers work similarly to Valgrind, but often are more efficient due to the face that unlike valgrind, they are not a purely runtime
tool. Depending on the tool, they claim to have a roughly `2x` slowdown.

These tools come with Clang and GCC, and work by simply adding compilation **and link** flags to your program,
and then running the program normally.

When using the sanitizers, they reccomend using optimization level 1 (flag `-O1`, that's an "oh" not a 0) which will disable
some optimizations (such as inlining) that can make analysis more difficult.

Some sanitizers (such as Asan (address sanitizer), UBsan (undefined behavior sanitizer), and Lsan (leak sanitizer)) can be run together
while others like Tsan (thread sanitizer) need separate builds.

### Address and Memory Sanitizer

Address sanitizer is similar to Valgrind's Memcheck. It works by applying the flag `-fsantizie=address`. For C++,
they reccommend also using the argument `-fno-omit-frame-pointer`. It may also be helpful to disable
tail call elimination with `-fno-optimize-sibling-call`.

Asan can also detect problems with initialization order by setting the environmental variable `ASAN_OPTIONS` to `check_initialization_order=1`.

Memory sanitizer checks for unitialized reads. Use `-fsanitize=memory`.

### Thread Sanitizer

This is similar to Valgrind's Helgrind and DRD. We enable this one by applying the flag `-fsanitize=thread`.
Thread Sanitizer checks for data races, and I'd reccomend this one over Valgrind if that's what you're
interested in.

Let's look at an example output:

```
WARNING: ThreadSanitizer: data race (pid=20495)
  Write of size 1 at 0x7b1000006959 by thread T10:
    #0 cmr::ASCIIMsgManager<...>::addMsgIdentifier(...) /.../msg_manager.h:158 (action_client_test+0x197ccd)
    #1 cmr::MsgManager::addMsgIdentifier(...) /.../msg_manager.h:35 (action_client_test+0x19402b)
    #2 cmr::msgFmt::ASCIIMsgFormatter::addMsgId(...) const /.../msg_formatter.cc:37 (action_client_test+0x19402b)
    #3 cmr::msgFmt::ASCIIMsgFormatter::nvi_formatReadMsg(...) const /.../msg_formatter.cc:47 (action_client_test+0x19402b)
    #4 ...
    #5 cmr::UartHal::readSensor<double, void>(...)::{lambda()#1}::operator()() const /.../uart_hal.h:201 (action_client_test+0x1a771f)
    ...
    #24 std::thread::_Invoker<std::tuple<std::function<void ()> > >::operator()() /usr/include/c++/9/thread:251 (action_client_test+0x1e7565)
    #25 std::thread::_State_impl<std::thread::_Invoker<std::tuple<std::function<void ()> > > >::_M_run() /usr/include/c++/9/thread:195 (action_client_test+0x1e7565)
    #26 <null> <null> (libstdc++.so.6+0xd44bf)

  Previous write of size 1 at 0x7b1000006959 by thread T9:
    #0 cmr::ASCIIMsgManager<std::atomic<unsigned char>, void>::addMsgIdentifier(...) (action_client_test+0x197ccd)
    #1 cmr::MsgManager::addMsgIdentifier(...) (action_client_test+0x19412a)
    #2 cmr::msgFmt::ASCIIMsgFormatter::addMsgId(...) (action_client_test+0x19412a)
    #3 cmr::msgFmt::ASCIIMsgFormatter::nvi_formatWriteMsg(...) (action_client_test+0x19412a)
    #4 ...
    #5 operator() /.../uart_hal.cc:151 (action_client_test+0x1a2cc7)
    #6 _M_invoke /usr/include/c++/9/bits/std_function.h:300 (action_client_test+0x1a2cc7)
    #7 std::function<void ()>::operator()() const /usr/include/c++/9/bits/std_function.h:688 (action_client_test+0x19c93c)
    #8 cmr::resultFromExceptions(std::function<void ()>) /.../uart_hal.cc:36 (action_client_test+0x19c93c)
    #9 cmr::UartHal::setMotor(...) /.../uart_hal.cc:156 (action_client_test+0x19cbbd)
    #10 cmr::UartHal::setMotorEffort(...) /.../uart_hal.cc:161 (action_client_test+0x19cce2)
    ...

  Location is heap block of size 64 at 0x7b1000006940 allocated by main thread:
    #0 operator new(unsigned long) <null> (libtsan.so.0+0x8e62e)
    ...
    #10 main /ws/src/Superproject/cmr_hal/test/messaging_thread_test.cpp:305 (action_client_test+0xb2f9c)

  Thread T10 (tid=20510, running) created by thread T6 at:
    #0 pthread_create <null> (libtsan.so.0+0x5fe84)
    #1 std::thread::_M_start_thread(...) <null> (libstdc++.so.6+0xd4765)
    ...

  Thread T9 (tid=20509, running) created by thread T6 at:
    #0 pthread_create <null> (libtsan.so.0+0x5fe84)
    #1 std::thread::_M_start_thread(...) <null> (libstdc++.so.6+0xd4765)
    ...

SUMMARY: ThreadSanitizer: data race /.../msg_manager.h:158 in 
cmr::ASCIIMsgManager<std::atomic<unsigned char>, void>::addMsgIdentifier(...)
```

I used `...` to cut out paths, function argument types, and parts of stacktraces to make this easier to read. 
The big thing to look at is the `SUMMARY` line Tsan gives you, which gives you the file and line number
of the data race.

Similar to DRD, it will give you the stacktrace of the access for each thread. However this tool will also give
you stacktraces for where the conflicting threads are created, and give you a memory location for the data
that it detected a data race for.

### Undefined Behavior Sanitizer

As the name implies, this one checks for undefined behavior. We can compile with the `-fsanitize=undefined` flag.
You might also want to use `-fsanitize=nullability` in conjunction.

### Leak Sanitizer

As the name implies, this one checks for memory leaks. To use this one, you guessed it, link with `-fsanitize=leak`.

---
For more information on Sanitizers, check out [clang's compiler documentation](https://clang.llvm.org/docs/index.html) and for more information on Valgrind, check out [their website](https://valgrind.org/).