# Mutexes and Locks
"Shared state is broken state"[^1].
If you have two threads reading from the same piece of memory, that's fine, but if you have one reading while another is writing you can get tons of issues.
Firstly, the reading thread might not read a completely changed value on an object (ie. halfway between changing an integer) then for larger data types like classes,
a reading thread might only read some new values (ie. say the first boolean and integer, but not the last string).
Therefore, we need a way to synchronize multiple data accesses on the same data.

First, what exactly is an object? The standard defines it as a "region of storage".
An object is stored in one or more memory locations and can consist of sub-objects.
For example, a struct is an object where each member is a sub-object (with possibly sub-sub-objects).
Adjacent bit-fields are their own object but share a memory location.
However, zero length bit-fields break up the adjacency and separate bit-fields into different memory locations.
Each variable is an object, and variables of fundamental or primitive types, no matter their size, take up exactly one memory location.
Multiple threads can safely and concurrently access different memory locations.
```C++
struct A {
    int a; //obj
    bool b; //obj
    char c; //obj
    unsigned long d : 24, e : 8; 
    //two objs in same mem location
    unsigned spacer : 0;
    char f : 4, g : 4; 
    //2 objs same mem
}
```

Each object has a modification order, or the sequence of reads and writes that occur to it.
All threads must agree upon the modification order of a single object, but they don't have to agree on the modification order of an object relative to another.

### Thread Local
Sometimes you don't want shared state.
For those times you have the `thread_local` storage modifier.
Thread local variables are each unique to their own thread, and must also be static.
You don't need to synchronize on thread-local storage because each thread has their own copy.

```C++
class Bar {
private:
    static thread_local int localInt;
public:
    void doFoo() {
        // perform a computation with localInt
        // no synchronization needed
    }

}
```

If you can, avoid shared state.
It slows down concurrent applications (now they must serially access the state) and is harder to reason about.

I won't dive into the details, but two architectures of concurrency without shared state are Actors aka CSP, and Blackboards.
For the actor model, think of each thread as its own actor. Each actor can only communicate to other actors via a one way messaging system.
As messages come, an actor processes it to completion and has an internal state machine. 
For the backboard model, think of a bunch of investigators working on a case.
Each thread can put information on the blackboard and all other threads can process the information and add more to the blackboard.
Information comes in any order, and the threads have no knowledge about each other.

## Mutexes
Of course things aren't always as peachy as being able to avoid shared state.
When those times come, you must enforce a modification order between objects.
If you don't, you have a *race condition* where one thread is racing another to access the state. 

One tool to avoid that are mutexes which stand for MUTually EXclusive. Mutexes basically enforce
serial access to pieces of data. Threads lock a mutex before executing instructions that cannot be completed by multiple threads
at once. Once a thread has obtained the lock, other threads that try to lock the mutex must wait until the
thread that has the mutex releases it. When a thread is done with a mutex, it must release it so other threads
can acquire it and execute the code the mutex protects.

In C++, we have `std::mutex`. I'm not even going to enumerate the member functions of a mutex because you should basically never use it directly.
Instead, you want to wrap it in a class such as `std::lock_guard<>` or `std::scoped_lock<>`.
A lock guard is the simplest RAII object for a mutex.
When it's constructed it locks immediately and won't unlock until its destroyed.
It's neither moveable or copyable, it simply guards a piece of code that uses shared state.
Its lack in functionality is made up for by being lightweight.

```C++
std::mutex mu;
std::queue<char> sharedState;

void workerThread() {
    while(/* do work */){
        std::lock_guard<std::mutex> lk(mu);
        if(!sharedState.empty()){
            auto c = sharedState.front();
            sharedState.pop();
            // process it
        }
    }
}
std::thread t1(&workerThread);
std::thread t2(&workerThread);
std::thread t3([&mutex, &sharedState](){
    while(/* do work*/){
        auto c = //compute c
        std::lock_guard<std::mutex> lk(mutex);
        sharedState.push(c);
    }
});
```
Now looking at the code, we see that in the worker thread, we process the data while still holding the lock. What if that takes a long time? Could we fix it by doing this?
```C++
void workerThread(){
    bool empty = false;
    {
        std::lock_guard<std::mutex> lk(mu);
        empty = sharedState.empty();
    }
    if(!empty){
        {
            std::lock_guard<std::mutex> lk(mu);
            //front and pop
        }
        // process as before
    }
}
```

Let's say that we take the lock and store the value `false` from the call to `empty()` into the local variable `empty`.
After releasing the lock, let's say that another thread pops off the last element of the stack.
Now when the first thread goes to pop an element off the stack, it will find that it's already empty!
In general, we want to use locks to make operations like this atomic, so a thread can't be interrupted when its doing this.
Moreover, in general we want to hold a lock for as narrow of a scope as possible, and as little time as possible.
However, as shown in the above example, we can't lock on too fine of a scale so that single operations can be interrupted by other threads.

`scoped_lock<>` is pretty much exactly like `lock_guard`, however `scoped_lock` allows locking multiple mutexes at the same time.
Generally, `std::scoped_lock` should be preferred to `std::lock_guard`.

```C++
std::mutex m1, m2;

{
    std::scoped_lock lk(m1, m2);
    // compiler deduces themplate type
    // full definitions is:
    // std::scoped_lock<std::mutex, std::mutex> lk(m1, m2);
}
```


 
Let's look at a more powerful lock. `std::unique_lock` can be unlocked and re-locked and it can be moved. 
```C++
std::unique_lock<std::mutex> lk(mu, std::lock_deferred);
// don't lock on construction
lk.lock(); //lock it.
//OR std::lock(lk);

if(!sharedState.empty()){
    auto c = sharedState.front();
    sharedState.pop();
    lk.unlock(); //unlock
    // process it
}
```

`std::lock()` allows locking two locks at the same time. 
Using `std::unique_lock`, we're able to easily lock the mutex,
check if the shared stack isn't empty and if it isn't, pop the top of the stack,
then finally unlock the lock to perform processing without inhibiting other threads
from processing elements separately.

You don't have to worry about forgetting to unlock a `std::unique_lock` since `std::unique_lock` will unlock itself during destruction.

```C++
std::mutex m1, m2;

std::lock(m1, m2);
// lock m1 and m2 atomically together
std::lock_guard<std::mutex> lk1(m1, std::lock_adopt);
// take over an already locked mutex
std::lock_guard<std::mutex> lk2(m2, std::lock_adopt);

// OR

std::unique_lock<std::mutex> lk3(m1, std::lock_deferred), lk4(m2, std::lock_deferred);
std::lock(lk3, lk4);

//OR even better

std::scoped_lock lk(m1, m2);

// lock on construction
std::unique_lock<std::mutex> lk5(m1);
```

Note: Although I show them all in the same code sample, you **cannot** lock an already locked mutex.
`std::mutex` is **not** reentrant.

Notice how these RAII objects are templates.
That sort of implies that there is more than one type of mutex. 

We also have the `std::shared_mutex`. 
A shared mutex allows threads to access data concurrently by using a `std::shared_lock`, and will only allow exclusive access using a `std::unique_lock`.
This is commonly referred to as a *reader-writer* locks because typically you have a group of threads safely reading data at the same time,
and one thread writing data exclusively. This is used in data structures where the usage pattern is to commonly read and infrequently update data.

```C++
/**
* Calculates an id based on the current msg count and 
* the thread id of the calling thread
* Since we're only working with 1 byte, there can 
* only be max 32 threads to one msg formatter 
* and only 8 writes before a read per thread
* @return unique id for the message
*/
uint8_t calcId(std::atomic<uint8_t>& counter) {
    static std::map<std::thread::id, uint8_t> threadIds;
    static std::shared_mutex threadIdMu;
    // shared mutex
    static uint8_t idCounter = 0;
    
    auto index = counter++ % 8;
    uint8_t id = index << 5;
    std::shared_lock lk(threadIdMu);
    // lock for reading

    auto it = 
        threadIds.find(std::this_thread::get_id());
    if (it != threadIds.end()) {
        // common case
        id |= it->second;
    }
    else {
        // rare case

        lk.unlock(); //unlock reading
        std::unique_lock lk2(threadIdMu);
        // lock for writing

        threadIds.emplace(std::this_thread::get_id(), 
           static_cast<uint8_t>(idCounter));
        id |= (idCounter++ & 0b11111);
    
    }
    return id;
}
```
As you can see, we use `std::unique_lock` to get exclusive access and write to the map and `std::shared_lock` to get shared access read from the map.
Shared locks have the same semantics as `unique_lock`, except for being shared access of course.
When a thread wants to get exclusive access, it blocks other threads from locking the mutex for either shared or exclusive access, waits for all in-progress operations to stop, then
performs the operation and unlocks th lock. Threads waiting for the exclusive lock get priority. So then what happens if we have more threads looking for exclusive access than shared access? 
Well, we can end up with a situation known as
*starvation* where threads continually acquire the exclusive lock leaving threads waiting for the shared lock for a long time. Thus, it's important such locks be used in situations
where shared access is the common case and exclusive access happens less frequently.

Earlier I mentioned reentrant locks. Let's see that in C++. C++ has `std::recursive_mutex` which allows one thread to lock the mutex multiple times
and requires the thread to unlock it the same amount of times it has locked it. Think of this as basically allowing nested locks within a single thread.

```C++
class Foo {
    std::recursive_mutex mu;
public:
    void stuff() {
        std::lock_guard<std::recursive_mutex> lk(mu);
        // do some stuff
    }
    void moreStuff() {
        std::lock_guard<std::recursive_mutex> lk(mu);
        // complex stuff
        stuff();
    }
}
```
We have some code reuse here, which is great. Unfortunately, if we were to use a normal `std::mutex`, this code would deadlock because when we call `stuff()` from `moreStuff()`,
we'd be waiting for a lock, yet we already have the lock!
Basically, the thread would be waiting for itself to give up the lock, yet it won't give up the lock because it's waiting for a lock.
`std::recursive_mutex` avoids this problem, and you see we can happily implement `moreStuff()` in terms of `stuff()`.

Speaking of waiting, as I mentioned earlier, when you lock a mutex, you wait for it to become available if you don't have it.
What if you don't want to wait if it's not available, or at least not wait too long? We can use a
`std::timed_mutex` or `std::recursive_timed_mutex`! When using these, our RAII wrappers will be able to be able to use the member functions `try_lock()`,
`try_lock_for()`, and `try_lock_until()`. After a call to such functions, we can query if we obtained the lock with the member `owns_lock()`.
This allows us to avoid blocking (`try_lock()`), or avoid blocking for long periods of time.
```C++
using namespace std::chrono_literals;
std::timed_mutex mu;

std::unique_lock<std::timed_mutex> lk(mu, std::lock::deferred);
lk.try_lock_for(500ms); //wait 500ms if not available

if(lk.owns_lock()){
    //exclusive access
}
```


### Once Flag

Let's say you need to call some code once. It seems overkill to have a boolean flag, and a mutex to protect it. That's why we have `std::call_once`.

```C++
std::once_flag once;

std::call_once(once, []() {
    //execute this once
});
// pass the once_flag and a callable object to std::call_once
```

### Deadlock

I mentioned deadlock before, it's basically where no progress is made because one thread is waiting for another, but that thread is waiting for the first thread. More formally, deadlock occurs when all of the following conditions are met:
* Bounded Resources - we have a limited number of resources that many threads may try to access.
* No preemptions - a thread must wait for another thread to finish before taking the resource. Basically, threads are not stopped or interrupted during the time that is has the resource.
* Hold and Wait - a thread holding a resource waits for another resource.
* Circular Waiting - there exists a cycle in a resource acquisition graph. For example, thread `A` waits for `B`, who waits for `C`, who waits for `A`.

Typically Bounded Resources and No Preemptions are not conditions we can do much about, so most strategies at avoiding deadlock aim to prevent Hold and Wait and Circular Waiting from occuring at the same time.

Here are some guidelines to avoid deadlock:
* Use `std::lock()` or `std::scoped_lock<>` if you need to acquire multiple locks together
* Avoid nested locks
* Avoid calling user supplied code when holding a lock. This could possible lead to nested locks, since we don't know what that code does
* Always acquire and release locks in the same order
* Use a lock Hierarchy

    A Hierarchical lock is one which enforces an ordering. You can only lock locks going up (or down) the hierarchy.
    You can google for a simple implementation of one.

Deadlock and proper locking granularity are very simple concepts to describe,
but takes practice in order to identify and write code that considers both issues.
Thus, I'm going to refer you to C++ Concurrency in Action for more information.

---
[^1]: Professor Myers