# Structs

Structs are actually the same thing as a class except that their members are public by default. 
However, structs convey the idea of a *data structure* [^1], or a type with its internals known. 
Generally speaking, when you want to organize pieces of information together, but not necessarily provide an abstraction or any encapsulation, a struct is probably the best tool in the toolbox. 

A lot of times structs will be PODs (plain old data) which means their members are laid out contiguously in memory as they are written. 
PODs are composed of primitive types, enumerations, and classes or structs with a trivial default constructor and trivial copy constructors (pretty much the compiler-generated ones). 
This makes it possible to read and write and entire struct as binary to a stream such as a socket or file provided you take things such as padding and endianness into account. 
This might not have made much sense, but don't worry about it right now.

If no constructor is user-defined, you can use braces to initialize the struct passing in values for each member, in the order that they are declared.


```C++
struct Address {
    int zipCode, streetNumber;
    std::string city, state, country, street;
};

Address addr {
    1000, 21,
    "Ithaca", "NY", "USA", "Buffalo Rd"
};

addr.zipCode = 2000;
// public members by default
```

Rules of thumb for choosing a class vs struct:
* If there is any encapsulation, use a `class`
* Use `class` if there are invariants that must be enforced. Use `struct` if members can vary independently.

[^1]: Using this term as Uncle Bob uses it. His idea is that a data structure has its internals public while an object doesn't. Operations on an object tell it to do something, and ones on data structures are told to get something.