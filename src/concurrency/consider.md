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
If it's not there, that's called a miss and finding the data would be a hit.
So as its looking for data the processor goes lower and lower in the hierarchy until it finds what it's looking for.
When it does, it doesn't just load the minimum amount of data it needs. Instead, we apply the principle of spatial locality which states that data stored
together is likely accessed together. There's also the principle of temporal locality which states that data defined together (near in time) is typically used together.
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
Consider what would happen if you are constantly mutating shared state.
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

## Memory Order

Store operations can have `std::memory_order_relaxed`, `std::memory_order_release`, or `std::memory_order_seq_cst`.
Loads can have `std::memory_order_relaxed`, `std::memory_order_acquire`, or `std::memory_order_seq_cst`.
RMW (read-modify-write such as `operator++`) can have `std::memory_order_relaxed`, `std::memory_order_acquire`, `std::memory_order_release`, `std::memory_order_acq_rel`, or
`std::memory_order_seq_cst`.

`seq_cst` is the strictest and default order and enforces a total modification order between threads.
That means that at the end of the day, there is some concrete order of events taking place that all threads recognize.
When an atomic with `seq_cst` memory ordering is set, all threads see the updated value. You can think of `seq_cst` memory ordering
as enforcing that all threads work off a single memory location.

Relaxed memory ordering is the weakest, and it basically allows processors to more or less only work off their local cache and synchronize their cache to main memory when they see fit.
During a relaxed load, the only constraint is that a thread will read a value at least as recent as the last value they know about.
Consider that thread `A` stores 23 to variable `a` and then 100 to `b`, and then thread `B` stores 42 to `a` and then 200 to `b`.
The next time `A` loads `a`, it might still be 23. However, it may load `b` and see the value is 200.
In fact, the next 100 times `A` loads `a`, it might still only read 23.
In thread `A`'s perspective, `b` was updated to 200 before `a` was updated to 42.

For thread `B` however, the next time it loads `a` or `b` it must get 42 and 200 respectively,
because it already knows about these more recent values. Thread `B` could not load 23 from `a` after
storing `42` to it because it already knows about the more recent modification.
In thread `B`'s perspective, `a` was updated to 42 before `b` was updated to 200.

Exchange operations in a relaxed memory ordering are a little stronger. Extending the cubicle analogy,
and exchange would be like telling the man to write down a certain number and give you the last number in the list.
An exchange will always read the most recent value available.
Similarly, a compare exchange operation would be like telling the man: "I think `x` is at the bottom of the list,
if it is, write down `y`, otherwise tell me what is at the bottom of the list."

```C++
std::atomic<bool> x = false, y = false;
std::atomic<int> z = 0;
void write() {
    x.store(true, std::memory_order_relaxed);
    y.store(true, std::memory_order_relaxed);
}
void read() {
    while(!y.load(std::memory_order_relaxed)); //wait for y to be true
    if(x.load(std::memory_order_relaxed));
        ++z;
}
int main() {
    //write() and read() on separated threads

    //z can be 0 because when y is true, no guaruntee x is true on the thread executing read()
    //  even though x is updated before y on the thread executing write()

    //in seq_cst memory ordering, z can never be 0
}
```

During an acquire-release operation, we store a value with a release memory ordering and load with an acquire memory ordering.
A release operation *synchronizes-with* an acquire operations that reads the same value the release operation stored.
Thus, any updates that occur to thread `A` prior to the release operation are available on thread `B` after the acquire operation.
This is the same idea behind barriers and semaphores.

Consider:
```C++
std::atomic<bool> x = false, y = false;
std::atomic<int> z = 0;
void write() {
    x.store(true, std::memory_order_relaxed);
    y.store(true, std::memory_order_release);
}
void read() {
    while(!y.load(std::memory_order_acquire)); //wait for y to be true
    if(x.load(std::memory_order_relaxed));
        ++z;
}
int main() {
    //write() and read() on separated threads

    /* z can not be 0 this time
    this is because the load from y synchronizes with the store to y
    any updates that are sequenced before the store to y become visible
    to the thread that loads from y

    Notice how we achieved the same results as seq_cst memory ordering
    with weaker (and more efficient) memory orderings
    */
}
```

Acquire/release memory ordering can be thought of like so:
Imagine that when we tell the man a value, we also give him a batch number that value is part of and let him know if that update is the last number in the batch.
Now when we load values, the man will tell us if the value he gave us was the last number in a batch, and if it is he'll inform us who gave him that batch along with the batch number.
Now let's call up a woman in another cubicle playing the same game. This time, we'll ask her for a number and inform her we know about batch number `X` from caller `Z`.
This time, not only does the woman have to give us a number more recent than the last number we loaded or stored from her, but she also has to give us a number at least as recent
as any value in batch `X` from caller `Z`.

Last thing to mention is something called a *release sequence*. If you have a release store, acquire load, and a sequence of RMW operations that happen in between the load and store,
this forms a release sequence. The RMW operations in between can have *any* memory ordering and the acquire will still synchronize with the final load.

```C++
void populate_queue() {
    //populdate data on queue
    count.store(numberOfItems - 1, std::memory_order_release);
}

void consume_queue() {
    while(true) {
        int index;
        if((index = count.fetch_sub(1, std::memory_order_acquire) <= 0) {
            //RMW -= operation

            //wait
            continue;
        }
        process(queue[index]);
    }
}
```

---
[^1]: Analogy from C++ Concurrency In Action, see book for more information