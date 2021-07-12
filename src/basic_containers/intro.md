# Containers

Containers are classes in C++ that, well, contain things. The C++ standard library is really good at adhering to the principle of "one word per concept". Meaning that containers, although there do not share a common base class, do indeed share a common static interface. For example, while different containers have different notion of size, the `size()` member function is always a `const noexcept` function that gets the size of the container. We don't have some classes with a `size()` function, others with a `length()` function, and still others with a `length` member variable. This consistency makes the STL pretty easy to use once you get the hang of it. In designing your own classes, you should also strive to stay consistent with this style. Even outside C++, "one word per concept" is a good rule of thumb for naming things, functions with different names should have different meanings. Staying consistent like this makes an interface easier to use, and easy to use interfaces are prone to less errors.

Furthermore, STL containers are templates. (They are part of the standard template library). This means that they aren't limited to containing only one type but instead can be instantiated on many different types.

## Iterators

We'll discuss iterators more in depth later, but STL containers define functions `begin()` and `end()` to return iterators to the beginning and end of the container respectively. An iterator must define, at minimum, an `operator*` to get the current element from the iterator, `operator++` to advance the iterator to the next element in the container, and `operator!=` to compare iterators for equality. There are actually different types of iterators, each with a set of functions they must implement. The iterator returned from `end()` is known as a *sentinel* and is typically an iterator the points to the slot off the back of the container. When iterating, an iterator is advanced, element by element, until it equals the sentinel, indicating that the iterator has reached an invalid element off the back of the container and iteration has finished.

The reason why iterators have this somewhat convoluted interface is because for many containers that are contiguous in memory, pointers are their iterators. Thus pointers must be valid iterators so we can't require all iterators to define something like a `next()` method.

When a class defines a `begin()` and `end()` method, we can use that class in an *enhanced for loop*. The compiler will manage the incrementing for us, and give us the actual value of the element to use in the body of the loop.

```C++
std::string name = "Kathy";
*name.begin() = 'H';

for (char c : name) {
    std::cout << c;
}
// prints "Hathy"

// same as:
for (auto it = name.begin(); it != name.end(); ++it) {
    char c = *it; // get the element the iterator "points to"
    std::cout << c;
}
```

Iterators are an area where you really want to use prefix `++` rather than postfix unless you actually need the old value. While iterators for containers will most likely be pointers moving along a buffer in memory, that's not always the case and using postfix `++` rather than prefix may cause excess copying.

Some containers allow iterating in reverse. This can be done by using the `rbegin()` and `rend()` methods. Using these reverse iterators, you would still use `operator++`, but that operator would cause the iterator to go backwards. Many containers also define `cbegin()` and `cend()` functions to get constant iterators which prevent you from mutating the element they point to. However, in order to enable iteration of `const` objects in an enhanced for loop, the container must define overloads of `begin()` and `end()` which are `const` member functions that return `const` iterators.

```C++
std::vector nums = {10, 20, 30, 40, 50};
auto numIt = nums.cbegin();
*numIt; // 10
*numIt = 20; // error, const iterator
std::advance(numIt, 3); // advance iterator by 3 spaces
*numIt; //40

for (auto it = numIt; it != nums.end(); ++it) {
    std::cout << *it << " ";
} // "40 50 "

for (auto it = nums.rbegin(); it != nums.rend(); ++it) {
    std::cout << *it << " ";
} // "50 40 30 20 10 "

for (auto it = nums.rbegin(); it != nums.begin(); ++it) {
    // BAD: mismatch iterators
    // must use corresponding begin and end methods
    // also must not use begin from two different instances of a container
    // or two different containers
}

for (auto num : nums) {
    std::cout << num << " ";
} // "10 20 30 40 50 "
```

## Emplace

The `emplace` series of member functions (`emplace()`, `emplace_back()`, `emplace_front()`, etc.) allows constructing an element directly in the container rather than copying one that already exists. The arguments of an `emplace` function are *forwarded* to the constructor. Therefore, any series of arguments that can be used in a constructor to construct a new object that the container contains can be passed to an `emplace` function. This also includes the move constructor. Therefore `emplace` is the method to use to move an element into the container rather than copy it. This allows us to have containers of move-only types such as `std::unique_ptr`. We'll discuss how this works later.

```C++
struct MyStruct {
    int a, b, c;

    MyStruct(int a, int b, int c) : a(a), b(b), c(c) {}
};

std::vector<MyStruct> structs;

structs.emplace_back(5, 20, 30);
// construct a new MyStruct with a = 5, b = 20, and c = 30 onto the back of structs
structs[0].c; // 30

MyStruct s2 = {0, 0, 0};
structs.emplace_back(std::move(s2));
// move s2 into the container by using the move constructor

MyStruct s3 = {1, 1, 1};
structs.push_back(s3); // copy s3 into the container
s3.a; //1 - s3 was copied so it's still valid to use
```