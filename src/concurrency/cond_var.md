# Condition Variables, Latches, Barriers, and Semaphores

In the previous section I showed a common multhithreading architecture, producer-consumer,
where we had some worker threads and producer threads.
Now in that section, the consumer threads spin locked,
constantly looping to check if there was something for them to do.
This wastes precious cycles and prevent something else from being done on that core while it's spinning.
This is exactly the motivation of a condition variable, which lets you put a thread to sleep while it waits for another thread to notify it and wake up.

```C++
std::condition_variable cv;
std::mutex mu;
std::queue</*something*/> q;

void workerThread(){
while(/* some flag*/){
    std::unique_lock<std::mutex> lk(mu);
    if(q.empty()){
        cv.wait(lk, [&q]() {
            return !q.empty()
        });
    }
    auto t = q.front();
    q.pop();
    lk.unlock();
    // process it
}
}
void producer() {
while(/* something */){
    auto t = //compute it
    std::unique_lock<std::mutex> lk(mu);
    q.push(t);
    lk.unlock();
    // not holding mutex when calling notify()
    cv.notify_one();
}
}
```
A few things to note. First, when we wait for the lock, we pass the lock and a predicate to `wait()`.
We must use a `std::unique_lock` because the condition variable will handle unlocking the lock for us.
A condition variable will release the lock allowing other threads to acquire it. When the thread wakes up, the condition variable
retakes the lock (waiting if another thread already has it) and then resumes execution from where it left off.
We pass a predicate to the `wait` function so that when the thread wakes up, it will execute the predicate.
If the predicate returns `true`, then execution resumes. Otherwise, the thread releases the lock and goes back to sleep.
You may be wondering: "wouldn't being woken up be enough to know that data is available?" 
The answer is no, because of *spurious wakeups* and *lost wakeups*.
A spurious wakeup is when a condition variable wakes up randomly, and not because it was notified. 
A lost wakeup is when a condition variable is notified before a thread waits upon it.
Thus, when the thread waits on the condition variable, it will never wake up because the notification was already sent.
Neither are faults of C++, but rather limitations of condition variables in the OS.
Thus, a predicate is passed which is checked before the thread waits on a condition variable and immediately after a thread wakes up.
When the predicate returns `true`, the thread can proceed from waiting. 
If the condition variable waits, it unlocks the lock and essentially goes to sleep. 
When it awakes, it acquires the lock again, and if the predicate is true continues on. Otherwise, it releases the lock and resumes sleep.
If, on the other hand, the predicate returns true before the condition variable puts the thread to sleep, the thread's execution
will just continue on as if the wait call was never there.
Passing the predicate is optional, but in most cases you want it.

The condition variable also has `wait_for()` and `wait_until()` and which allow you to pass `std::duration`s or `std::time_point`s to wait for a specified amount of time
or wait until the system clock reaches a certain time.

Now for the producer thread.
Notice we release the lock before calling `notify_one()`. 
If we didn't, then the consumer thread might awake to find the lock is still held, and thus must block until the lock is released.
This is just good practice to avoid premature pesimization. We also call `notify_one()`, which as the name suggests wakes up one thread.
There is also a `notify_all()`, which awakes all waiting threads.

There's a slight problem with this implementation.
What if the worker thread throws when processing the data from the queue?
Well the data is already popped off the queue, and no other thread will be able to process it. Here's a modification we can make to avoid this.

```C++
{
    auto t = q.front();
    q.pop();
    lk.unlock();
    try{
        //process
    } catch(...) {
        {
            std::scoped_lock lg(mu);
            q.push(t);
            // t should have a nothrow copy
            // if not, wrap it in a smart pointer
            // and have the queue hold smart pointers
        }
        cv.notify_one();
        
    }
}
```
There are nicer ways to do this (for starters, this probably should be broken up into multiple
functions because it's doing way more than one thing), but this is the basic idea.

A `std::condition_variable` only works with `std::unique_lock`. 
To do this with a `shared_lock` or another type, use `std::condition_variable_any`.
Other than that the interface is the same.

## Latches and Barriers

These are new to C++20, but they can be accessed in C++17 as part of the `std::experimental` namespace.

A barrier is something that stops the sequence of execution for a thread, until multiple threads reach it.
Once a thread reaches a barrier, they can drop themselves from the barrier (barrier no longer will wait on them).
Once x threads reach the barrier, the barrier releases and resets.

```C++
std::barrier barrier(3); //3 threads must wait
int result[3];
void doWork() {
    while(true){
        //do some work
        result[id] = //result.
        barrier.arrive_and_wait();
        // barrier.arrive(); don't wait
        // barrier.arrive_and_drop(); decrements wait count
        int res = result[/*some other id*/];
    }
}
```
doWork() can be run on 3 different threads. Each thread will perform a computation, update its result,
and wait for the others. Notice something interesting: result is unprotected shared state.
Why is this ok? Well, I'm not going to talk much about memory ordering, but I will say this:

An instruction that is *sequenced-before* another on the same thread of execution (so, comes first top-to-bottom) *happens-before* the second instruction.
The barrier provides a mechanism to *synchronize-with* another thread. So if `A` happens-before `B`, and `B` synchronizes-with `C`,
and `C` happens-before `D`, then `A` *inter-thread-happens-before* `D`. 
Thus, utilizing data from `A` in `D`, is safe. There's more to this story but that's good start.
So when a thread arrives at the barrier, all the updates that happened on that thread before the barrier are synchronized to all other threads ater the barrier.
Let's say thread `A` assigns its result to `result[0]`. Then `A` reaches the barrier and waits. When thread `B` and `C` arrive at the barrier,
the wait count will reach `3` and all the threads will be released and continue execution.
During this process, the threads synchronize the modifications made to `result` so that all threads
can see the results stored in `result` prior to the barrier.
Now thread `C` is safe to read `result[0]` from thread `A`.
If however, `A` made another update to `result[0]` after it resumed from the barrier, then `C` might read the old value (the one set before the barrier),
the new value, or something in-between.

A latch is exactly like a barrier, but it cannot be reset.

```C++
std::latch latch(4);
void worker() {
    //do work
    latch.count_down();
}

//spawn 3 workers which call worker()
//do some work of its own
latch.arrive_and_wait();
// counts down latch and waits for count to be zero
// now access results of workers

// we also have
latch.try_wait() 
// which tests if the count is zero
```

### Semaphore

A semaphore is essentially a counter.
Unlike mutexes, it doesn't have to be accessed from a single thread of execution.
When using mutexes, you can't unlock a mutex you don't have, but a semaphore is just a counter.
Thus, one thread can acquire the semaphore and another thread can release it. 
A semaphore has two main operations: wait (acquire) and signal (release). Acquire decrements the semaphore's count.
If the count is already 0, it will wait until it can decrement the count. Release will do the opposite,
it will increment the semaphore which will unblock acquire.
Like the others, you can use `try_acquire` or `try_acquire_for` to avoid waiting if the count is already 0.

A `std::binary_semaphore` is a special case of a semaphore (`std::counting_semaphore`) where the count only goes up to 1.

```C++
std::counting_semaphore<20> sem(0); 
//initialize count to 0 with a least max value of 20
std::deque q;
void worker() {
    while(true){
        sem.acquire();
        // decrement count
        // synchronize recent changes from the producer thread

        auto item = q.front();
        q.pop_front();
        // process
    }
}
void producer() {
    while(true){
        //compute data
        q.push_back(/*data*/);
        sem.release();
        // increment count
    }
}

std::thread t(&worker);
producer();
sem.max(); //get semaphore max value
```
Notice how the semaphore is essentially acting as a counter for the elements in the queue.
Unlike mutexes, the semaphore doesn't have to be released on the same thread that it's acquired.
As a template parameter, the semaphore takes the least max value, essentially the smallest maximum value the semaphore will count up to.
A binary semaphore is just a type alias for `std::counting_semaphore<1>`.

Similar to the barrier, each corresponding `acquire()` and `release()` match up so that modifications on a thread prior to the call to `release()`
will be visible on another thread after the call to `acquire()`. Therefore, `std::deque` does not need a lock protecting it.