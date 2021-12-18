# Move Semantics

Take this example:

```C++
std::string makeGreeting(const std::string & personName) {
    return "Hello, " + personName + " it is very nice to meet you!";
}

std::string greetJake = makeGreeting("Jake");
```

In pre C++11, `"Jake"` would be constructed into an `std::string`, and each `operator+` would also construct new `std::string`s. 
Next the `return` statement could copy construct a temporary `std::string` which is returned from the function and `greetJake` could be copy constructed from that temporary. 
A lot of copies! 

Without changing any of the code and simply compiling with a modern version, the last two copies become (relatively) low cost moves. 
As we saw earlier, moves often amount to just copying pointers. However, as we discussed this isn't always the case. 
One such exception are strings, due to SSO (small string optimization) the move will also have to copy the small stack allocated buffer which is typically <= 15 bytes in size. 
Therefore, moving a string amounts to copying a pointer, integral size, and the small buffer while moving something like an `std::vector` would typically only amount to copying a pointer and integral size.

This story isn't quite over, so please save your cards and letters, but I hope you can see the motivation for moving.

## Move Basics (from constructors chapter)

Now I talk about this move constructor. But what is moving exactly? Well, instead of performing a copy, we move the internals from one object to another, leaving behind an empty shell of an object to be destroyed. 
When the old object is destroyed, nothing happens because the object's internal state has been moved out and put into a new object.

The double ampersand is an *rvalue reference*, and basically it is a reference to temporary values. 
For example, the return value of a function is moved (well, sometimes, more on this later) into a temporary value and that temporary is returned. 
The temporary gets destroyed at the end of the statement that called the function. We can also manually move non-temporaries with the `std::move` function. 
However, once a value has been moved, you **must not** use it again since all of its state has been put into a different object.

Now frankly, I've told you a flat out lie, but we'll discuss this in way more detail later.

```C++
std::string getString() {
    return "Hello";
}

std::string greet(std::string && name) {
    return "Hello " + name;
}

std::string myStr = getString(); // move constructor
std::string myStr2 = std::move(myStr); // move constructor again
const auto myStr3 = myStr2; // copy constructor

std::string myName = "Jacob";
auto greeting = greet(std::move(myName));
    // move ctor for name in the greet() function
    // move ctor for greeting
```

---

> "Premature optimization is the root of all evil."
>
> \- Donald Knuth