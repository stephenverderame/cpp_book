# Async, Promises, and Futures

Earlier I mentioned that threads take void return functions.
Well, they don't have to, but returning something from a thread isn't going to do anything useful.
That isn't the case for futures. Futures are essentially a handle to a concurrent task that allows you to wait for and fetch the result of the task. The basic operations of futures are:
* `get()` - Blocks until the result is ready, then returns the result
* `valid()` - Returns true if the future is associated with shared state
* `wait()`, `wait_for()`, and `wait_until()` 
    These functions return a `std::future_status` which indicates if the future is ready (can call `get()` without blocking) or not

Futures are returned from `std::async` which may run a task in parallel.
When using `std::async`, the library controls if the code is run on a new thread or sequentially. The library implementation chooses the best option to avoid over-subscription and excessive task switching.
If you must force it to run in parallel, pass `std::launch::async` as the first parameter.
The next parameter to `std::asycnc` would be the callable object to run, and any remaining parameters would be arguments passed to that callable object.
This works similarly to `std::bind`. `std::async` will then return a `std::future`, which is like a handle for a value that isn't available yet, but is expected
in the future.

```C++
int foo();
class A {
public:
    int computeNumber(int seed) {
    int num = //...
    return num;
    }
};
A a;
int seed = 20;
std::future<int> fu = std::async(&A::computeNumber, &A, seed);
// if you pass a member function, the first argument needs to be a pointer
// to the owning object (remember, this is an implicit first argument)

// do some stuff

int res = fu.get();
// if async did not launch a parallel task
// then at this point it will execute the code 
// in sequence either way, get() will wait 
// until the result is available

std::vector<std::future<int>> futures;
futures.emplace_back(std::async(std::launch::async, [](){
    return /* some result*/
}));
// emplace a task that is always run on a new thread

futures.emplace_back(std::async(std::launch::deferred, &foo));
// std::launch::deferedd forces the task to be run on the current thread

for(auto& fu : futures) {
    auto status = fu.wait_for(500ms);
    if(status == std::future_status::ready) {
        int res = fu.get();
    }
}
```
Like threads, futures are move-only.
So notice when we iterate through the vector, we use `auto&` instead of `auto`.
This is because, if you remember our type deduction rules, a plain `auto` is exactly like a plain `T`.
Therefore, it will copy by-value. Futures cannot be copied (nor did we want to copy it), so we use `auto&` to get a reference to the deduced type.

We also have `std::launch::deferred` which is diametrically opposed to `std::launch::async`.
Instead of requiring a parallel mode of execution, it requires execution be deferred until `get()` is called.

`get()` can **only be called once** per future object. Once `get()` is called, when the result is ready, the future moves out whatever shared state it had.
At this point it becomes invalid.
You can check if a future is valid by calling `valid()`. If you do call `get()` twice on the same future, you will get a `std::future_error`.
If a future is destroyed without calling `get()`, the destructor will wait for the task to finish.

`std::async` is great for recursive algorithms, times when you don't know how best to split up work among different threads ahead of time,
or using functional programming idioms with concurrent programming.

Here's and example of a parallel merge sort.

```C++
#include <future>
#include <algorithm>
template<typename T>
void mergeSort(std::vector<T>& vec, size_t begin, size_t end) {
    if (end > begin + 1) {
        const auto middle = (begin + end) / 2;
        auto fu = std::async([&]() { mergeSort(vec, begin, middle); });
        mergeSort(vec, middle, end);
        fu.get();
        std::inplace_merge(&vec[begin], &vec[middle], vec.data() + end);
    }
}
```
What's going on is that we start with the entire vector, and at each step we subdivide the vector into two halves.
One half is (possibly) performed in parallel while the main thread is doing the other half.
If we used pure threads here, then we could quickly get up to too large of an amount of threads.
We start big, and go into smaller and smaller chunks so that the new threads are likely created for the big chunks,
and the smaller chunks are likely executed sequentially.

Notice also that the merge sort function doesn't return anything and performs the operation in-place. We can still use futures however.
In this case we use a future to notify when the task is completed instead of using condition variables and mutexes.
Consider using void futures to notify one-off events.

Since `mergeSort` is a template, we wrap it in a lambda and pass it to `std::async` instead of passing its address and function arguments directly.
This is because it's difficult to get the address of a template function.

Since `end` is going to be off the back of the vector for some calls to `std::inplace_merge`, we don't get the end pointer by doing `&vec[end]`, because
that would be undefined behavior if `end >= size()`. Finally, I'd also like to remind you that doing `&vec[index]` will not work for boolean vectors.
For an explanation, refer to [the vectors section in the basic containers chapters](../basic_containers/vector.md).

Now what if you wanted to be able to call `get()` multiple times?
Let's say you have multiple threads that all need some result.
Well the C++ standard has given us the shared future `std::shared_future`. The differences are quite similar to a `unique_ptr` vs a `shared_ptr`.
The `shared_future` is both moveable and copyable.
We can convert a future to a `shared_future` by calling the `share()` member function.
Once shared, multiple threads can then wait on the result and multiple threads can call `get()`.

```C++
std::shared_future<X> fu;
void doWork(){
    auto val = fu.get();
    // do something
}
fu = std::async(std::launch::async, &getResult).share();
std::thread t1(&doWork), t2(&doWork);
```
Other than that the interface is exactly the same.

### Continuations
The `std::experimental::future` provides another member function `then()`.
This allows you to schedule some code to run when the future becomes ready instead of having to check it yourself by calling `get()` or `wait()`.
This is known as a *continuation*. It takes any kind of callable object, similar to bind. `std::experimental::when_all()`
allows you to pass a variable amount of futures, and it will return a future that becomes ready when all the futures are ready.
Furthermore, `std::experimental::when_any()` does a similar thing, except its future will become ready once any of the passed futures are ready.

### Packaged Task

std::async isn't the only way to create futures.
We also have `std::packed_task`. 
Packaged task associates a callable object with a future.
The `packaged_task` is manually executable via `operator()`.
Essentially, it allows you to package together a callable object and its associated future, then move it somewhere to be called.
So the main difference compared to `std::async` is that you control when, where, and how it's executed.
Consider this basic thread pool, which is something that maintains multiple worker threads that the user can use to complete tasks.

```C++
class ThreadPool {
private:
    std::queue<std::packaged_task<int(void)>> q;
    std::vector<std::jthread> thread;
    std::condition_variable cv;
    std::mutex mu;

    // function that is run on each worker thread
    // of the thread pool
    void process(std::stop_token stop) {
        /* while thread isn't interrupted:
            wait for new task if queue is empty
            acquire lock
            pop task of work queue
            release lock
            run task that was popped of the queue
        */
        while(!stop.stop_requested()){
            std::unique_lock<std::mutex> lk(mu);
            if(q.empty()){
                cv.wait(lk, [&q](){
                    return !q.empty();
                });
            }
            auto pt = std::move(q.front());
            q.pop();
            lk.unlock();
            pt(); //execute packaged_task
        }
    }
public:
    // Creates a pool with tCount amount of worker threads
    ThreadPool(int tCount) {
        for(auto i = 0; i < tCount; ++i){
            threads.emplace_back(
                std::jthread(&ThreadPool::process, this));
        }
    }

    std::future<int> addWork(std::function<int(void)> f) {
        std::packaged_task<int(void)> pt(f);
        // construct packaged_task from functor

        auto fu = pt.get_future();
        // get associated future
        {
            std::lock_guard<std::mutex> lk(mu);
            q.push(std::move(pt));
            // push new task to complete on queue
        }
        cv.notify_one();
        // notify worker threads that there's a new task
        return fu;
        }
    }
}'

int fibo(int val) {
    if(val <= 1) return val;
    return fibo(val - 1) + fibo(val - 2);
}

ThreadPool pool(std::thread::hardware_concurrency());
// create a thread pool with the amount of threads the system can support truly concurrently
auto future = pool.addWork(std::bind(&fibo, 345));
```

First, we use the new C++20 jthread. This will ensure that threads are cleaned up properly, and if a thread throws while being created the threads that started already are cleaned up correctly.
It also allows us to interrupt the thread. Our thread pool doesn't provide that feature to the clients, but jthread will interrupt on destruction before joining.
We call `get_future()` on the packaged task to get the associated future and return it to the client so that they can receive the result.
Since packaged tasks are move only, we have to use `std::move` to explicitly move it.
We use a condition variable to notify the thread pool of new work to do, and a `unique_lock` to control access to the shared queue.

### Promise

There's one final way to create a future, and that's from a promise. A promise is like the writing end of a future.
You can call `set_value()` or `set_exception()` on a promise and that information will be relayed to the future.
These functions will make the future ready. `set_value()` will set the data returned from calling `get()` on the future.
`set_exception()` will set the exception thrown when `get()` is called on the future.
If an error occurs during the call to `set_value()`, then an exception is set on the promise.
The promise **must outlive the future** because the promise is what owns the state, not the future.
If the promise gets destroyed before the future gets the data, then the exception `std::future_error::broken_promise` is set instead.
When you call `get()` on a future, the data owned by the promise is moved to the future and returned to the client.
You can get a future from a promise by calling `get_future()` just like packaged tasks.


```C++
std::promise<std::string> p;
void compute() {
    try{
        std::string s = //some calculation
        p.set_value(s);
        // if the copy fails here
        // the promise automatically sets an exception
    } catch(...){
        p.set_exception(std::current_exception());
        // captures whatever exception was thrown and
        // stores it in the future
    }
}

std::thread t(&compute);
auto fu = p.get_future();

std::cout << fu.get() << std::endl;
```
Just like how you can't call `get()` multiple times on a future,
you can't set a value multiple times on the same promise.
Think of it as a one-direction pipe from promise to future which will close after you send a message on it. 

I'd also like to point out `std::current_exception()`, which I think needs no further explanation than my comments