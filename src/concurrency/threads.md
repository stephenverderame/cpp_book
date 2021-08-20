# Threads
 
A thread is a parallel mode of execution. 
Whether it's truly parallel or not is a different story but from a programmer's perspective, things on two separate threads can occur at the same time. 
C++11 provides `std::thread`. Every joinable thread (we'll discuss that in a minute) has a thread id which can be used to identify threads. 
The id can be copied, stored and compared with other thread ids.
The actual implementation is abstracted, so do not assume it is an integer.
To get it you can call the member function `get_id()` or `std::this_thread::get_id()` to get the id of the current thread your code is running on.
 
A thread is constructed similarly to a bind object. It takes a callable object (including member function pointers) followed by any arguments to call it with and executes that right away on a new thread.

```C++
std::thread t([]() {
auto cur id = std::this_thread::get_id();
// this code runs at the same time
});
t.get_id();
// as this code

A a;
std::thread t2(&A::workerThread, &a);
// note a must outlive t2
```

A thread can be joined or detached. When you detach a thread,
you sever the connection with it, and let it run in the background.
Most modern OSs will clean them up when your application terminates.
When you join a thread, you wait for it to finish before moving on.
A thread is said to be *joinable* if it has neither been detached nor joined previously, and it was not default constructed (so it was provided with something to run). 

Let me pose a question for you. What do you think should happen when a joinable thread is destroyed?
Should the application wait for the thread to finish or detach the thread?
If you said "not sure", well that's basically the answer the C++ Standard Committee concluded.
If you join on destruction, then there's the possibly this could cause horrible hidden blocks in you code as your code must wait for the thread to finish.
If you detach it on destruction then there's almost the certain possibility than any outside state it uses (say things captured in a lambda) will be destroyed.
At best this will look like an invalid memory access, and the code will throw, at worst this will look like some garbage or half garbage
(maybe only the last byte in the object is reused) leading to broken invariants and hard-to-find bugs.
So what happens? Well the program terminates. The Committee took the "dead programs tell no lies" approach.
Thus, you must make sure a thread is **unjoinable on all paths**. If this sounds like "performing some action no matter what", then you might already be thinking RAII. 
```C++
void myFunc();

std::thread m(&myFunc);

if(m.joinable())
m.detach(); // sever ties with thread

class jthread {
private:
    std::thread t;
public:
    template<typename ... Args>
    jthread(Args&& ... args) {
        t = std::thread(std::forward<Args>(args)...);
    }
    
    jthread() = default;
    // default constructed thread is not joinable 
    // (no thread is started)
    
    
    ~jthread() {
        if(t.joinable())
            t.join();
    }
}
```
Threads are move only. This intuitively makes sense. How could we make a unique copy?
We could start a new thread using the same callable object,
but a user might not expect such behavior and not make their callable object safe to be run on two independent threads.

Threads are limited resources. Earlier I used the term "truly" concurrent because that hardly ever happens.
First off, your computer can only support a number of truly concurrent operations to begin with.
On your CPU this is normally your "core count". However even this is a bit of a fudged number sometimes. Intel has something they call "virtual cores"; they duplicate some common components in each core, but not all of them.
This allows certain stages of the pipeline to occur at the same time, but not every stage.
To get the amount of threads supported concurrently, use `std::thread::hardware_concurrency()`. 

Moreover, Amdahl's Law states:
$$\frac{1}{(1 - p) + \frac{p}{c}} \leq \frac{1}{1 - p}$$
Where
\begin{align*}
p &= \mbox{ fraction of parallelizeable work} \\
c &= \mbox{ number of cores} \\
\end{align*}
Thus lets say our application spends 20% (quite a good chunk) of its time doing parallel work, then it can only be improved by a factor of:
$$\frac{1}{1 - 0.2} = 1.25$$
no matter how many threads or cores you have.

There's more to the story, but my main point is don't overuse threads.
When you have much more threads than your hardware can handle, you get what's called *oversubscription* and *task switching*.
To switch tasks, the processor has to save all the state of the current thread, store it, and load up the next thread and state of that.
This can take a lot of time and if it has to do this excessively... no bueno.

My point is threading is powerful, but it's not to be used everywhere.

### JThread

C++20 adds a jthread class, which joins on destruction ("joining thread"). It also allows the thread to be interrupted via a `stop_token`.
I will say no more, but show a code sample from cppreference. 
```C++
using namespace std::chrono_literals;

void f(std::stop_token stop_token, int value)
{
    while (!stop_token.stop_requested()) {
        std::cout << value++ << ' ' << std::flush;
        std::this_thread::sleep_for(200ms);
    }
    std::cout << std::endl;
}

int main()
{
    std::jthread thread(f, 5);
    std::this_thread::sleep_for(3s);
    // The destructor of jthread calls
    // request_stop() and join().
    
    
    thread.request_stop() // call manually
    auto ss = thread.get_stop_source();
    ss.request_stop(); //requests stop
    bool b = ss.stop_requested();
    bool c = ss.stop_possible();
}
```