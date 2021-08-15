# Unique Pointers

Unique pointers, and really anything else that's "unique" in C++, model exclusive ownership. 
They offer practically no space or time overhead as compared to raw pointers. Because they model exclusive ownership, they cannot be copied, only moved. 
Here's a bad, quick, and dirty example of how something like this might be implemented.

```C++
template<typename T>
class BadUniquePtr {
    T * data;

    void freeData() {
        if (data) {
            delete data;
            data = nullptr;
        }
    }
public:
    BadUniquePtr(T * owningPtr) : data(owningPtr) {}
    BadUniquePtr() : data(nullptr) {}

    BadUniquePtr(const BadUniquePtr &) = delete;
    BadUniquePtr& operator=(const BadUniquePtr &) = delete;

    BadUniquePtr(BadUniquePtr && other) : BadUniquePtr() {
        swap(data, other.data);
    }

    BadUniquePtr& operator=(BadUniquePtr && other) {
       freeData();
       swap(data, other.data);
    }

    ~BadUniquePtr() {
        freeData();
    }
}
```

`std::unique_ptr` is a template, and so when you instantiate it with a type it replaces every template argument (`T` in `BadUniquePtr`) with whatever type you are instantiating it with.

`std::unique_ptr` provides the `reset()` member function to manually free the data or swap the internal pointer with a different one. 
If we need to, we can get access to the internal pointer with `get()` or `release()`. The latter returns the pointer and releases it from the `unique_ptr`s management. 
We can also swap the owning data between two `unique_ptr`s with the `swap()` **member function**. `std::unique_ptr` also provide `operator*` and `operator->` so they can be de-referenced just like raw pointers.

```C++
auto unique = std::make_unique<double>(6.28);
double* ptr = unique.get(); // the unique ptr is a class itself
// to access the unique_ptrs members use the dot operator
// to access the underlying data's members use ->

//ptr is not an owning pointer

auto u2 = std::make_unique<double>(3.14);
unique.swap(u2);
// Smart pointers must be pointers to the same type to swap

double* owningPtr = u2.release();
// u2 no longer manages the data

unique.reset(); // free data and set to nullptr
// which is the same as
unique = nullptr;

auto u3 = std::make_unique<double>(2.67);

unique = u3; // error, cannot copy
unique = owningPtr; // takes ownership of pointer

unique.reset(new double(-1.12)); // free data and take ownership of new raw ptr


// ---

auto getPtr() {
    return std::make_ptr<long long>(19474579);
    // move (Pre C++17), good
}

auto u4 = getPtr(); // move ctor, good
auto u5 = std::move(u4); // move ctor, good

u4 = std::move(u5); // move assignment, good
```

Smart pointers can store arrays as well. 
They handle calling the correct delete too. A smart pointer wrapped around an array overloads `operator[]` to provide index access. 
While this is slightly better than unmanaged arrays, an `std::vector` would be better because the smart pointer still does not store the size of the array.

A pointer of an array is a pointer to the first element in the array since arrays are stored contiguously in memory. 
A C string is a `char *` (or `const char *`) that is an array of characters with the last one being the null terminator ('\0' which is 0).

```C++
std::unique_ptr<char[]> string = std::make_unique<char[]>(100);
// 100 length char array
string[0] = 'h';
string[1] = 'i';
string[2] = '\0';
*string; // 'h'
// pointer is to the first element of the array

const char * cStr = string.get();

std::cout << cStr; // "hi"
```

# Deleters

This is all well and good if we want to use C++'s `delete` or `delete[]`. But what if we have a custom allocator or want to use C style allocation. 
Well luckily, smart pointers abstract away how they delete the data and delegate that responsibility to a deleter. 
The default deleter uses `delete` or `delete[]` depending on the type it wraps (`delete` for `T`, `delete[]` for `T[]`). The actual definition of `std::unique_ptr` is

```C++
template<typename T, typename Deleter = std::default_delete<T>>
class unique_ptr // ..
```

In reality there is a second template argument! This type should be the type of a callable object that overloads `operator()` to accept a pointer to the underlying data. 
`operator()` would then be responsible for performing the logic to cleanup the passed pointer. 
With a custom deleter, we cannot use `std::make_unique` because this function abstracts away the usage of the default deleter's matching allocation functions. 
Thus, we must use the `unique_ptr` constructor, passing a pointer as the first argument and an instance of the deleter type as the second.

```C++
struct Foo {
    int num;

    Foo() : num(0) {}
}

struct FooDeleter {
    void operator()(Foo * ptr) {
        ptr->~Foo();
        free(ptr);
    }
}

Foo * makeFoo() {
    const auto mem = malloc(sizeof(Foo));
    return 
        new (mem) Foo(); // placement new, calls constructor and constructs object
        // in already allocated memory
        // once again, more on this later

}

std::unique_ptr<Foo, FooDeleter> fooPtr(makeFoo(), FooDeleter());
// use fooPtr like normal

fooPtr->num = 100;
```

Now this seems like a bit of boilerplate just for two function calls. Well, we'll explain the following soon enough, but here's some other ways of doing the same thing:

```C++

void freeFoo(Foo * ptr) {
    ptr->~Foo();
    free(ptr);
}

std::unique_ptr<Foo, std::function<void(Foo*)>> fp2(makeFoo(), [](Foo * ptr) {
    ptr->~Foo();
    free(ptr);
});

// Use a lambda as the callable object
// passes a Foo* and returns void

std::unique_ptr<Foo, void(*)(Foo*)> fp3(makeFoo(), &freeFoo);
// function pointer is the deleter, pass free directly
// pointer to a function that takes a void* and returns void

std::unique_ptr<Foo, decltype(&freeFoo)> fp4(makeFoo(), &freeFoo); 
// same as fp3, just slightly easier since function pointer syntax is a pain

std::unique_ptr<Foo, std::function<void(Foo*)>> fp5(makeFoo(), &freeFoo);
// same as fp3 and fp4 but instead use a generalized function as the type
```

Here's another example of a custom deleter that adds some housekeeping to the standard C++ allocation.

```C++
// There's a few issues with this, but this is mainly to demonstrate deleters
/// Invariant: contains pointers to active allocations or nullptr to indicate freed memory
std::vector<Bar*> allocations;

auto makeBar() { // strong
    allocations.push_back(nullptr); //may reallocate and move entire vector (more on this later), strong
    auto ptr = new Bar();
    const auto idx = allocations.size() - 1; // no throw
    allocations[idx] = ptr; //copying pointer cannot throw
    const auto deleter = [idx, &allocations](Bar * del) {
        delete del;
        allocations[idx] = nullptr;
    }; // noexcept (construction of lambda and deleter itself)
    return std::unique_ptr<Bar, std::function<void(Bar*)>>(ptr, deleter); //noexcept
}

auto barPtr = makeBar();
```

By the way, deleters work the same for `shared_ptr` too