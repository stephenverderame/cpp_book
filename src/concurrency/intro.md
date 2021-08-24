# Concurrency

Concurrent code can execute non-sequentially, or out of order. When something is concurrent, that **does not** imply that it will occur *in parallel*,
meaning at the exact same time as something else. *Parallelism* is when code executes *in parallel*, sometimes referred to as truly concurrent.

To explain the difference, let's look at the following example:
```C++
int foo(int x) {
    int a = 10 * x * x;
    int b = 20 * x;
    int c = 30;
    return a + b + c
}

execConcurrent([]() { return foo(2); }); // task A
execConcurrent([]() { return foo(4); }); // task B
```
Consider we run this code on a system with a single logical CPU core. There is no hardware to support executing this code in parallel,
but we can still execute the code concurrently. From a programmer's perspective, the two functions may start and end at the same time,
but on a hardware level, there is no support in this example for the system to execute two instructions at once, or a single instruction
on multiple pieces of data. So the system might execute the code like this:

```
A calls foo(2)
B calls foo(4)
A assigns A_a to 10 * 2 * 2
B assigns B_a to 20 * 4
A assigns A_b to 20 * 2
A assigns A_c to 30
B assigns B_b to 20 * 4
A computes the sum
B assigns B_c to 30
B computes the sum
B returns the sum
A returns the sum
```

This is concurrency because instructions were not executed sequentially for a given program; instructions of two tasks were interleaved.
However, it is not parallelism because instructions were executed linearly and one at a time.

When using the concurrency API, there's no easy way to know if code is running in parallel, or just concurrently.
And truthfully, it doesn't matter.