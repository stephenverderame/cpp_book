# Atomics

Do you remember how I said we can have a situation where an integer is half-updated?
Well that's because normal read and writes are not atomic.
When something is atomic, it is basically an indivisible operation. An atomic operation is either fully completed, or not started.
Which means it forces some kind of modification order between threads. For most primitive types, atomic data types are lock free,
that is they are implemented using purely atomic operations provided by the ISA (Instruction Set Architecture, think of this as the API the hardware provides to software).
This isn't always the case and sometimes they are implemented via locks. You can call the `is_lock_free()` member function to determine this at runtime.
The `is_always_lock_free` static constant will tell you at compile time if the atomic is lock free on all supported architectures for the compiler.

One atomic type that is **always** lock-free is `std::atomic_flag`.
It's extremely primitive and only offers two functions: `clear()` and `test_and_set()`. `clear()` sets the flag to "false", and `test_and_set()`
will return the last value of the flag and set it to "true". That's it.
Now, as simple as it is, this is enough to provide a synchronizes-with relationship between two threads.
Remember, a synchronizes-with relationship allows any updates prior to the synchronization point on thread `A` to be
visible to thread `B` after the synchronization point.

```C++
std::atomic_flag flag;
void processor() {
    while(true) {
        if(!flag.test_and_set()){
            auto val = data.front();
            data.pop_front();
        }
    }
}
void producer() {
    while(true) {
        data.push_back(/*data*/);
        flag.clear();
    }
}
```
This is very similar to our barrier example.
The processor will never read data from the front unless the flag is false.
The producer will never set the flag to false unless there is data on the queue.
Thus, it is perfectly safe to access the queue without a lock.
We say that such synchronization forces each thread to recognize the changes made in the other prior to the synchronization.
In this case the processor must recognize changes the producer made to the queue prior to the producer clearing the flag.
I'll talk about this more in the next chapter, but each core has their own cache.
Synchronization like this forces a core to update their cache with the new data from another processor's cache.
This is how objects can have different relative modification orders to each other according to different threads.
Let's say thread `A` updates value `a` to 3, then 5, and 6. Then thread `B` updates value `b` to 20, then 30 then 40.
Both threads will see `a -> 3 -> 5 -> 6` and `b -> 20 -> 30 -> 40`.
However, the relative ordering (ie. `a -> 5` before or after `b -> 20`) will differ depending on the memory ordering used. 
I'm going to only discuss the strictest (and default) memory ordering: sequentially consistent but there are others.
Sequentially consistent memory ordering enforces a total ordering.
So with `std::seq_cst` we can ask if `a \to 5` happened before `b \to 20` and get an answer.
On other orderings such an inquiry is moot because each thread will have a different answer to that question.

So let's look at some more useful atomic types. The most basic operations on atomic types are `store()`, `load()`,
`exchange()`, `compare_exchange_strong()`, and `compare_exchange_weak()`. `store()` and `load()` are rather self-explanatory.
`exchange()` will return the old value and store a new value that's passed to it as a parameter. We call these RMW (read-modify-write) operations.
`compare__exchange`. isn't going to be very important to our discussion here, but they are vital for the development of lock free data structures. 
`compare_exchange_weak(expected, desired)` will compare `expected` with the value of the atomic, if they are equal, then `desired` will be stored.
If they are unuequal, `expected` will be updated with whatever is currently stored in the atomic.
Equality is tested by byte-wise comparisons such as `memcmp`. It **will not** use any form of overloaded `==` if your type provides own.
Given this, floats and doubles may lead to `compare_exchange` returning improper results due to slight differences in representation of the same number.
`compare_exchange_weak()` will also fail if the exchange cannot be guaranteed to be atomic.
This happens if the ISA does not have an atomic `compare_exchange` instruction, and the scheduler interrupts the thread.
This is known as a *spurious failure*.
`compare_exchange_strong` is guaranteed to return `false` only if expected was not equal to the current value in the atomic.
Internally, it uses a loop on `compare_exchange_weak`. `std::atomic<>` is a template class. Here's the same flag implementation, but this time with `std::atomic<bool>`.
```C++
std::atomic<bool> flag = false;
//initialize to false
void processor(){
    while(true) {
        if(flag.load()){
            auto val = data.front();
            data.pop_front();
        }
    }
}
void producer() {
    while(true){
        data.push_back(/*data*/);
        flag.store(true);
        flag = true;
        // overloaded operator=()
    }
}
```
Because atomic variables, are, well atomic, no locking or synchronization mechanism is needed to access them from multiple threads.
```C++
std::atomic<int> count = 0;
void processor(){
    while(count.load() < 500){
        ++count;
        // overloaded ++
    }
}
std::thread t1(&processor), t2(&processor);
```
This isn't a particularly useful code sample, but it does show how numerical types
have `++` and `--` both postfix and prefix versions overloaded. Instead of `++` and `--`, we also have the functions `fetch_add()`, `fetch_sub()`, `fetch_and()` etc.
Numerical atomics also overload `+`, `+=` and other arithmetic functions.

Finally, atomic also has free functions for each member function.
For example `std::atomic_load()`. They are specialized for each primitive atomic type and also for `std::shared_ptr`.
However, now there is `std::experiemental::shared_ptr` and in C++20 we can use `std::atomic<std::shared_ptr>>`.

In C++20, we also have the `std::atomic_ref`.
The interface is exactly the same as any other atomic type, but it is constructed from an existing lvalue reference.
Once an `atomic_ref` is created, access to that object must be exclusively through `atomic_ref` instances. 

Here's a very basic example
```C++
template<typename T>
class lock_free_stack
{
private:
    struct node
    {
         T data;
         std::atomic<std::shared_ptr<node>> next;
         node(T const& data_): data(data_) {}
    };
    std::atomic<shared_ptr<node>> head;
public:
    void push(T const& data)
    {
        auto const new_node = 
            std::make_shared<node>(data);
        new_node->next = head.load();
        // another thread could change head here
        while(!head.compare_exchange_weak(new_node->next, new_node));
        /* 
        while head != new_node->next:
            update new_node->next to be head
        when head == new_node->next, set head to be new_node
        */
    }
    void pop(T& result)
    {
         auto old_head = head.load();
         while(
            !head.compare_exchange_weak(old_head,old_head->next));
            result = old_head->data;
    }
```
When we push something, we first set the new node's next pointer to whatever the head is.
Then, we keep checking to make sure `head` hasn't been updated by checking that our new node's next pointer is still the head pointer.
If it is, then `head` is atomically set to `new_node`.
If `head` has changed and is no longer `new_node->next`, then we update `new_node->next` and try again.

To pop from the stack, we follow a similar process.
We first get a copy of head.
Then, as long as `head` hasn't changed, we atomically set `head` to whatever it is pointing to next.
If `head` has changed, then we update `old_head` and try again.
Notice we have an output parameter.
This is an over generalization, but generally speaking for pop operations such as this we have two options.
Return by reference via output parameter or make the user pass a smart pointer. The reason is that a copy operation can throw.
Moreover, when we return by smart pointer (whose copy cannot throw), we could not allocate the smart pointer in `pop()` because the allocation can fail.

`std::atomic<shared_ptr<T>>` does not make accesses of the underlying data thread safe, but it makes changing the actual pointer value thread safe. 

Lock free data structures are generally hard to design and even harder to get right, so only go for it if it is **necessary**.
They're even harder when we relax the memory ordering and start using relaxed or acquire-release memory ordering instead of sequentially consistent.
As far as I know however, x86 only supports sequentially consistent memory ordering.

Here's a common issue with lock-free data structures known as the *ABA* problem. Consider:
1. Thread 1 reads a value `A` from a variable. Then Thread 1 does something to `A` (such as dereference it).
2. Thread 1 is stalled
3. Thread 2 performs an operation changing the variable's value to `B`. Another thread does something based on the new value `B` so that it invalidates `A` (such as freeing `A`)
4. Another thread then changes the value back to `A` based on the new data (such as a new pointer with the same address)
5. Thread 1 resumes, uses a `compare_exchange` to find that the value is still `A` (although now it's a different `A`), and continues on with invalid data.