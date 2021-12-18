# Advanced Templates

In this chapter we'll look at some more generic programming techniques, along with some more ways we can run computations at compile time.

Many compilers have a *step-limit*, which roughly correlates to how many times they'll recurse before stopping compilation.
Compile-time computations are recursive, and this step-limit ensures that compilation doesn't take too long.
Most compilers have an option to increase the step-limit if you need to.

## A Word about Generic Programming

Let's talk about generic programming for a bit. The goal of generics is to define an implementation in terms of types that we don't yet know. Contrast that to OOP where we do know the type, but we don't know the implementation. It accomplishes many of the same goals as OOP, but in a different way. I want to first define a few terms that I'll use without explanation:

* **static interface** - An interface is an interface, the term is mostly agnostic to programming paradigms and means the same in OOP and here. However I use the term *static interface* to explicitly refer to a common interface shared by different classes. The static interface is essentially the concepts our generic algorithms require of the generic types passed to them.
* **static polymorphism** - Once again, polymorphism is polymorphism in OOP and Generics. However when I say *static polymorphism* I explicitly refer to the way in which we can pass different classes that uphold the same static interface to a generic algorithm.

Here's an example comparing OOP and Generics:

```C++
// This is our interface
class Vehicle {
public:
    virtual ~Vehicle() = default;

    virtual void transport() const = 0;
}

// subtyes of our interface/implementations of interface
class Car : public Vehicle {
    //...
}

class Plane : public Vehicle {
    //...
}



// polymorphic usage
// any class that inherits from Vehicle can be used
void travel(const Vehicle& vehicle) {
    vehicle.transport();
}


// C++ 20 implementation so it looks closer to OOP
// this is our interface
template<typename T>
concept Animal = requires(T t) {
    { t.makeSound() } -> std::same_as<void>;
}

//implementations of interface
class Bird {
public:
    void makeSound() {/*...*/}
}

class Dog {
public:
    void makeSound() {/*...*/}
}


// polymorphic usage
// any class that defines a void makeSound() function can be used
template<Animal AnimalType>
void animalSound(const AnimalType& animal) {
    animal.makeSound();
}
```

I use C++20 concepts to make it easier to see the similarities. However the same idea applies even if you use C++17 techniques
or a simple requires clause in the spec of `animalSound()`. OOP incurs runtime cost of the extra memory for the virtual table,
extra indirection for dynamic dispatch, and extra indirection for use of pointers. Templates however, slow down compilation and
typically lead to larger binary sizes due to the need for the compiler to generate a new class/function for every type it's
instantiated with. Sometimes, templates have to be used if you want to merge two class heirarchies that you cannot modify.
For example, you might be using two libraries, each with their own heirarchy of vector/matrix classes. 

In this chapter, we'll look at more techniques and strategies for template programming.

> "The keys to an engineer's heart are acronyms. Lots and lots of acronyms."
>
> \- Me, 2021