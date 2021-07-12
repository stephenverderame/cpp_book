# Shared Pointers

While unique pointers model exclusive ownership, shared pointers model shared ownership. 

The shared pointer uses reference counting to keep track of the amount of other pointers referencing a piece of data. During the constructor, the reference count is incremented and during destruction the count is decremented. When that reference count goes to 0, the data is cleaned up. Because of this reference counting ability, shared pointers do have an added space and time overhead. Unlike a unique pointer, a shared pointer doesn't hold a pointer directly to the data but instead has a pointer to shared state containing the underlying data and meta data such as the reference count. Furthermore, the shared pointer's reference counting logic is atomic and thread safe. This does not mean it synchronizes accesses to the data, but it does guarantee that only one thread will every attempt to free the data. The atomicity of the reference counting operation also incurs some extra overhead since atomic instructions are typically more expensive then non-atomic ones. 

Unlike unique pointers. Shared pointers are copyable. During a copy the pointer to the shared state is copied and the reference count in this shared state is incremented. The old data's reference count is also decremented.

Shared pointers have an interface that's extremely similar to unique pointers but with some differences. You can use the `use_count()` member function to get the current data's reference count. Since a shared pointer models shared ownership, there is no `release()` member function since this could free data out from under other shared pointer instances. Finally, we can move a `unique_ptr` into a `shared_ptr` via its constructor to start shared management of exclusively owned data. This makes `unique_ptr` great for factory functions since it's easily convertible to any type of pointer you may want to use.

```C++
auto ptr = std::make_shared<int>(20);
auto ptr2 = ptr; // copy, increment reference count
*ptr2 = 50;
*ptr; //50

ptr.use_count(); //2

std::vector<std::shared_ptr<Foo>> foos;

{
    auto p = std::make_shared<Foo>();
    // use p
    foos.push_back(p);
    // p goes out of scope here
    // no problem though because we copied the pointer and incremented the reference count
    // so the data in the vector remains valid
}

std::shared_ptr<Bar> barShared = std::make_unique<Bar>();
// upgrade from unique_ptr
auto b2 = barShared; // copy ctor
```

# Weak Pointers

Consider this linked list implementation

```C++
class LinkedList {
    struct Node {
        std::shared_ptr<Node> next, prev;
        int data;

        Node(int data, std::shared_ptr<Node> prev = nullptr) : 
            next(nullptr), prev(prev), data(data) {}
    }
    std::shared_ptr<Node> root;
public:
    /// ...
}
```

When creating recursive data types, we must use pointers otherwise the data type would have infinite size. But there's a problem with the implementation above. When we try to delete a node, we'll find that there's another reference to it via the next node's `prev` pointer. Therefore, the reference count will drop from 2 to 1, and the memory will not be freed. When we go to delete the next node. We'll find theres an existing reference in the previous node's `next` pointer since the previous node was not freed yet. Once again the reference count will not drop to 0.

This means that we cannot use `shared_ptr` in cyclic situations like the one above! This would create a memory leak. If we'd like to retain copy semantics, we can't use a `unique_ptr` so that leaves us with a `weak_ptr`.

A `weak_ptr` holds a non-owning reference to data managed by a `shared_ptr`. The difference is that during copy of a `shared_ptr`, the reference count is incremented while operations of `weak_ptr` do not touch the reference count at all. This means that `weak_ptr` may dangle! The last `shared_ptr` may go out of scope, destroying the underlying data while a `weak_ptr` is still active!

This isn't a problem however because a `weak_ptr` cannot access the underlying data directly. Instead, it must use the member `lock()` which will return a `shared_ptr` to the underlying data if it still is valid, or a default constructed `shared_ptr` for the underlying data. Like a `shared_ptr`, you can also check the amount of active owning references with `use_count()` and you can also check if the underlying data is still valid with the `expired()` member function. Finally, you can pass a reference to a `shared_ptr` to the `weak_ptr` constructor to construct a non-owning reference to the shared pointer's underlying data.

```C++
std::vector<std::weak_ptr<Foo>> foos;

{
    auto fPtr = std::make_shared<Foo>();
    std::weak_ptr fRef = fPtr;
    fPtr.use_count(); // 1
    fRef.expired(); // false;

    auto fPtr2 = fRef.lock();
    fPtr.use_count(); // 2
    auto fRef2 = fRef; // copy
    fPtr2.use_count(); // 2
    foos.push_back(fRef2);
} // data goes out of scope here

foos[0].expired(); // true

```

Using a `weak_ptr`, we can break cyclic references of `shared_ptr` and easily model non-owning references.


```C++
auto personFactory(std::string && name, int age) {
    return std::make_unique<Person>(std::move(name), age);
    // unique_ptr is great for factory functions
}
std::vector<std::shared_ptr<Person>> people;
std::vector<std::weak_ptr<Person>> pplRefs;

{
    std::shared_ptr<Person> ps = 
        personFactory("Bill", -1);
    // unique_ptr rvalue can be converted into a shared_ptr
    // this is because rvalues are temporaries and the
    // unique_ptr is being destroyed.
    // Since unique_ptrs model exclusive ownership, take means
    // it's safe to change the ownership model of the data

    people.push_back(ps);
    // ps can be copied and the internal data will 
    // persist past the scope

    pplRefs.emplace_back(ps);
    // shared pointers can be converted to weak ptrs
}

//after ps has gone out of scope
//use of the -> scope resolution operator to get access 
// to the object's functions
people[0]->getName();
//still works, data still persists

// use of . scope resolution operator to use methods of 
// the smart pointer itself
if(!pplRef[0].expired()){
    // check if data is destroyed or not
    std::shared_ptr<Person> person = pplRefs[0].lock();
}

//later...

throw std::runtime_error("Something bad");

// all smart pointers are cleaned up properly
```

### Possible Exercises

1. Create a doubly linked list of integers (or a template if you desire) using only smart pointers with the following interface
    * `push_back`, `push_front`, `pop_front`, and `pop_back` in `O(1)` time with the strong guarantee
    * `size()` member function in `O(1)` with no throw guarantee
    * `empty()` convenience function in `O(1)` that's `noexcept`
    * `find()` - may return a `bool` or an index. If you go for an index, you may use `-1` to indicate it doesn't exist in the list or you may return an `std::optional`. Strong
    * `erase()` - may take a value or index. Should do nothing if the element doesn't exist. Strong

2. Create a reference counted pointer RAII class that is non-atomic. The implementation should match `std::shared_ptr` (the points we talked about, you don't have to make every single constructor, overload, or non member). [cppreference](https://en.cppreference.com/w/cpp/memory/shared_ptr) may help.