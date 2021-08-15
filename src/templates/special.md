# Template Specialization

Earlier I talked about how the behavior of `std::vector` changed when it stores `bool` and how the behavior of smart pointers change when they point to arrays. 
This is possible due to template specialization, which basically provides a separate definition of a class or function used when a template parameter is a specific type.

Let's start with *total template specialization* which is when you want unique behavior for a specific combination of template arguments.

```C++
template<typename T, typename U>
class Foo {
    // ...
};

template<>
class Foo<int, long> {
    // ...
};

Foo<int, int>; // first
Foo<long, int>; // first
Foo<std::string, Person>; // first
Foo<int, long>; // second
```

A specialization does not automatically share the interface of the main definition (like for example public inheritance) and the interface for a template specialization may be completely different. 
However, in most cases where you use this, you'll probably find it most useful if the specialization and template class mostly share an interface.

Here's a more useful example. Let's say you had two classes, `Address` and `Person`, and to have a hash map with `Address` keys and `Person` values. 
You could manually add a `hash()` member function and key the map on that hash type, but then you'd have to manually ensure that `hash()` doesn't create collisions. 
Plus, you'd be doing extra computation as `std::unordered_map` would then hash that key again. Instead, we can specialize `std::hash` which is a `struct` that uses `operator()` to compute the hash.

```C++
struct Address {
    int zipCode, streetNumber;
    short state;
    std::string city, street;

    bool operator==(const Address& other) const {
        // ...
    }
};

class Person {
    // ...
};

namespace std {
    // std::hash is part of the std namespace so we must
    // put the specialization in that namespace as well

    // normally you shouldn't modify the std namespace
    // but this is an example that's ok

    template<>
    struct hash<Address> {

        size_t operator()(const Address& addr) const {
            const static std::hash<int> iHash;
            const static std::hash<short> sHash;
            const static std::hash<string> strHash;

            return iHash(zipCode) ^ iHash(streetNumber) ^
                sHash(state) ^ strHash(city) ^ strHash(street);
            // not much though went into this example hash function
        }
    };
}

std::unordered_map<Address, Person>; // good!
```

Functions may be specialized as well. However, generally, it's best to avoid specialization and prefer overloading the function unless you really need to specialize it. 
One scenario where you would need specialization is to have functions that differ in only their return type. This is because you cannot create overloads based on return type alone.

```C++
template<typename T, typename U>
void foo(const T& t, const U& u) {
    // ...
}

template<>
void foo<int, int>(const int& t, const int& u) {

}
```

During name resolution, the compiler will first pick the best overload, then pick the best specialization. 
This can lead to confusing results because a better match created by specializing a worse matching overload won't be picked over a decent matching overload.

```C++
template<typename T>
void foo(T f) {
    // ...
}

template<> foo<int*>(int* f) {} // specialize foo<T>

template<typename T>
void foo(T* f) {} // overload

foo(new int()); // calls foo(T*) not foo<T> specialization
// chooses best overload, then looks for specialization (which there are none)
```

However, changing the definition order so that the specialization specializes the second `foo` makes it work as expected.

```C++
template<typename T>
void foo(T f) {
    // ...
}

template<typename T>
void foo(T* f) {} // overload

template<> foo<int*>(int* f) {} // specialize foo<T*>

foo(new int()); // calls specialization
// chooses the T* overload since its a better match
// then sees the int* specialization
```

Classes don't overload. However, what we can do with classes (and not functions) is *partial template specialization*. 
Using this, we can create a specialization that is used when some, but not all template arguments match, or create specializations on other template types.

```C++
template<typename T, typename U>
class Foo {

};

template<typename T>
class Foo<T, std::string> {

};

template<typename T, typename U>
class Foo<std::unique_ptr<T>, U> {

};

Foo<int, int>; // #1
Foo<int, std::string>; //#2
Foo<std::unique_ptr<std::string>, long>; //#3
Foo<std::unique_ptr<int>, std::string>; // error, ambiguous specialization. Could be 2 or 3
```

You can inherit from templates and specialization as well. When inheriting from a template, you must use the `this` pointer to refer to the base class's members.

```C++
template<typename T>
class Foo : private std::vector<T> {
    // ...

    Foo() {
        emplace_back(); // error
        this->emplace_back(); // good
    }
};
```

### Possible Exercises

1. Specialize the `Ringbuffer` class so that when used with `bool`, it only takes up one bit per element.

2. Make the vector template from previous exercises hashable.