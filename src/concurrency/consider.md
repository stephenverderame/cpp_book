# Multithreading Considerations

As I alluded to earlier, multhithreading isn't all peachy, not even getting into the extra difficulty of writing multithreaded code.

### Memory Hierarchy Basics

The memory hierarchy is a design to optimize the tradeoff between speed and cost in memory.
Disk space is really cheap and we can have a lot of it.
But the downside is that it's really slow.
To overcome this computers have a memory hierarchy.
At the top we have registers. These are typically a few words in size (32 - 64 bits) and are used for direct computations by the processor (adding, subtracting, etc.).
They're right at the heart of the action so to speak. Below that we have cache. Cache is typically divided into 3 levels L1 to L3 where L1 is smaller and faster and L3 is larger and slower.
Cache is also on the processor but slower to access and farther away but much bigger than registers (L3 will typically be a few MB, L1 a few KB).
After that we have main memory or RAM. Following that we have non-volatile storage (will retain state after power loss) such as disk space, SSD, nVMEs etc.
When a processor looks for a value, it first looks at the highest level in the hierarchy and moves down.
If it's not there, that's called a miss and typically because this occurs in the cache, a cache miss \index{cache miss}. (Finding the data would be a cache hit\).
So as its looking for data the processor goes lower and lower in the hierarchy until it finds what it's looking for.
When it does, it doesn't just load the minimum amount of data it needs. Instead, we apply the principle of spatial locality which states that data stored
together are likely accessed together. There's also the principle of temporal locality which states that data defined together (near in time) is typically used together.
Thus, when the cpu needs to swap out data in its cache for data in main memory, it does so chunks at a time.
Furthermore, typical architectures implement a *write-back* policy. This means that an updated block in a higher level of the hierarchy is only stored back in the lower levels of the hierarchy
when that block is replaced.

This is how we can have different relative modification orders between threads.
To be more efficient, threads won't synchronize with main memory unless they have to.
Using relaxed memory ordering or acquire/release will allow threads to just work off the data in their own cache.
Thus when thread `A` asks for value `b` from thread `B`,
a relaxed memory ordering will allow the thread to just go to its cache and see whatever value it has there.
This may not be the latest `b`, but at one point it was `b`.
To put this another way, imagine there is a man in a cubicle with a phone. People use the phone and tell him to write down numbers, which he does, keeping a sequential list of values.
People can also use this phone to query numbers. The man must give each person the last number they received or any number that was written down after that number. Thus, even though the man may have
more recent numbers, he only has to give each caller a number at least as recent as the last number they know about. [^1]
Acquire-release memory ordering creates synchronization points between matching acquire and release calls similarly to a semaphore. 

It's also important to note that not all ISA's support these differences in memory orders.
x86 for example, doesn't have much different between sequentially consistent and all the order memory orderings.
The atomic operations used at the ISA level are essentially the same.

If you are considering relaxing memory orderings: first get it right with sequentially consistent ordering, then see if changing the ordering will reap a reward.
Remember to use profiling tools to see where bottlenecks are.


### Cache Ping Pong
Now as for how this affects you in your day to day programming: consider what would happen if you are constantly mutating shared state.
When using sequentially consistent memory ordering (or even relaxed memory ordering if these are RMW operations), and a thread updates some data,
a global modification order must be enforced. Thus, the data must be written back to main memory and loaded by the other threads.
This is a slow process and if it's happening constantly this can really slow down your program.

### False Sharing
Remember how data is read in chunks from main memory? Consider what would happen if each thread's data is close together.
Every time you read or write something, you'd have to go to main memory, get the latest data from the other threads and load it back into your cache because
each thread's data is read and written together. This can cause a lot of *contention*. C++ provides us with `std::thread::hardware_destructive_interference_size()`
 which returns the maximum sized `cache line` which is how much data is read at once into the cache.

### Oversubscription
I kind of mentioned this earlier, but basically a computer only has a limited amount of cores.
That number is the maximum amount of threads that can occur truly in parallel.
Anything more than that requires *task switching* which is when the cpu must save the state of the current thread and resume execution on another.
This just sounds slow, and it is!

## Execution Policies

Many STL algorithms now have an overload where the first argument is its execution policy.
No matter the execution policy, all algorithms will have slightly different semantics when a policy (even the sequential one) is specified.
If an exception other than `std::bad_alloc` is thrown, they will terminate the program and if the algorithm normally guarantees a running time of `N`,
then the execution specified version guarantees `O(N)`. There are 3 policies:
* Sequenced Policy (`std::execution::sequenced_policy`)
    Forces sequential operation on a given thread. 
    Mandates a set order of operations, but this order may differ from the normal non-policy-specified version.
    Can rely that operations are performed on the same thread, but cannot rely on any given order.
* Parallel Policy (`std::execution::parallel_policy`)
    Operations performed on the same thread must not be interleaved and must have a definite ordering.
    Operations cannot cause data races, they must protect any shared state via synchronization objects.
    This is a good choice for introducing parallelism.
* Parallel Unsequenced Policy (`std::execution::parallel_unsequenced_policy`)
    Operations may be interleaved on the same thread and between different threads.
    Operations **cannot** use any synchronization objects.
    Operations may start on one thread and finish on a different one.
    Good for things that don't have any shared state.


To use a policy, pass an instance of the policy class to an algorithm.
```C++
std::vector v = //..
std::for_each(std::execution::par, v.begin(), v.end(), 
    [](auto& elem) {
        elem += 10;
    });
```
Pass `std::execution::par` for parallel, `std::execution::seq` for sequenced and `std::execution::par_unseq`, for parallel un-sequenced.

---
[^1]: Analogy from C++ Concurrency In Action, see book for more information