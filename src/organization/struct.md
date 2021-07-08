# Structs

Structs are actually the same thing as a class except that their members are public by default. However structs convey the idea of a *data structure* [^1], or a type with its internals known. Generally speaking, when you want to organize pieces of information together, but not necessarily provide an abstraction or any encapsulation, a struct is probably the best tool in the toolbox. 

A lot of times structs will be PODs (plain old data) which means their members are layed out contiguously in memory as they are written with no extra stuff. PODs are composed of primitive types, enumerations, and must have a trivial default constructor and trivial copy constructors. This makes it possible to read and write and entire struct as binary to a stream such as a socket or file provided you take things such as padding and endianness into account. This might not have made much sense, but don't worry about it right now.

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

[^1]: Using this term as Uncle Bob uses it