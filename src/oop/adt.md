# Interfaces

What we saw last chapter was a *concrete* class. Unlike concrete types, abstract types cannot be instantiated, but instead provide an interface for other concrete classes.

In Java and other languages, all functions of a superclass may be overridden. This isn't the case in C++. A function that you want a derived class to be able to override must be declared `virtual`. The reason is that the presence of a `virtual` function requires something known as a virtual table or vtable. This is an area in memory that each instance has that allows the compiler to perform dynamic dispatch by looking up at runtime the specific function to call. Since C++ is "pay for what you use", this vtable isn't created unless a class declares a function `virtual` which enabled dynamic dispatch for that function and allows subclasses to override it.

An abstract type contains at least one *pure virtual* functions, which is a function that subclasses **must** override because the superclass does not implement it. The Java equivalent of an `interface` would be a class that contains only pure virtual functions.

```C++
class Person {
public:
    virtual std::string speak() const = 0; // pure virtual function

    virtual int walk() { return 10; }
    // overrideable with default implementation
};

class Child : public Person { // child implements Person
public:
    std::string speak() const override {
        return "Hiya";
    }
};

Child c;
c.speak(); //Hiya
c.walk(); // 10
Person p; //error
```

When overriding a function, it's good practice to explicitly denote it as such with `override`. This prevents you from accidentally creating a new function or *shadowing* the super class's functions. You cannot overload superclass functions. Creating a function with the same name as a superclass's function in a subclass shadows that function and prevents it from being called.

```C++
class Machine {
public:
    long serialNumber() { /* ... */ }
};

class Computer : public Machine {
public:
    long serialNumber(long default) { /* ... */ }
};

Computer pc;
pc.serialNumber(10); // good
pc.serialNumber(); // error, superclass method shadowed
```

Inheritance enables us to use polymorphism and let subclasses behave like their superclasses. However, in order to fully support this, we need to ensure that the destructor is virtual. Otherwise, there will be no dynamic dispatch on the destructor which will cause undefined behavior if we destroy a subclass through a reference or pointer to the superclass. Constructors on the other hand, should not be virtual.

As an addendum, you should not call virtual functions in constructors or destructors. This is because the actual type of the object changes during these two operations and the type is always the type being created/destroyed and never a subclass. It is technically safe so long as the virtual function being called is not pure virtual and you don't expect it to dispatch to a derived type, but it's not a good idea.

```C++
class Vehicle {
protected:
    std::string name;
public:
    Vehicle(const std::string & name) : name(name) {}

    virtual ~Vehicle() = default;
    // need a virtual destructor, but don't need any custom behavior
    // so mark it = default

    std::string description() { return name; }

    virtual void move() = 0;
};

class Car : public Vehicle {
private:
    int horsePower;
public:
    Car(int hp) : Vehicle("Car") {
        // cannot set variables in initializer list
        // when using a delegating constructor
        horsePower = hp;
    }

    Car(const std::string & name, int hp) : Vehicle(name) {
        horsePower = hp;
    }

    void move() override {
        // name is protected so it can be accessed
        std::cout << name << " went " 
            << horsePower / 10 /* some calculation, idk */ 
            << " mph";
    }

    void honk() {/* ... */}
};


auto moveIt(Vehicle & vehicle) {
    vehicle.move();
    return vehicle.description();
}

Car c(700);

moveIt(c);

```

`Vehicle` has a constructor, but it is an abstract type so it still can't be constructed. Instead subclass constructors must delegate one of the superclass constructors like shown. When delegating a call from one constructor to another, we cannot use the initializer list to instantiate extra variables.

Also notice how `moveIt` takes a `Vehicle` by reference. This is paramount to avoid *object slicing*. In memory, a concrete derived class is formed by taking its state and tacking it on to the data of the superclass. 

|&Car|
|---|
|Vehicle Data|
|`std::string name`|
|Car Data|
|`int horsePower`|

Thus, passing the superclass by value will only copy the data that the subclass shares with the superclass. Hence the name object slicing, because the data specific to the subclass is sliced off. One such piece of data is the subclass's vtable. Therefore, object slicing prevents dynamic dispatch from operating as expected. In this case, if `Vehicle` was not a reference, the compiler would complain since that would require it to construct a new instance of `Vehicle`, and `Vehicle` is abstract and cannot be constructed.

However, even with references we can run into a bit of a conundrum:

```C++
Car c1(300), c2("car 2", 500);
Vehicle& v = c1; //constructor, good
v = c2; //operator=
v.move(); // ?
```

As we saw in the last chapter, `operator=` is not normally virtual. We can make it virtual and overload it ourselves, but by default its not. Therefore, the above example causes object slicing as well! This is because `operator=` gets called on the static (declared) type which is `Vehicle` and not the dynamic (actual) type which is `Car`. So here, `operator=` will copy only the members it knows about (the ones part of the static type) and leave the rest unchanged. Thus, the output of `v.move()` is "car 2 went 30 mph".

## Covariance and Contravariance

Suppose we had a function `clone` in our previous hierarchy. In the interface, we might define it as follows:

```C++
    virtual Vehicle& clone() = 0;
```

But in `Car`, we know that if we clone a `Car` we'll get another `Car` back, so it would sure be nice to override it like such:

```C++
    Car& clone() override {/* ... */ }
```
Well we actually can! This is known as *covariant* return types and it permits derived classes to return an object that is derived from the return type of the virtual function. This is not an overload and in fact, two functions that differ in **only** their return types are not considered overloads.

Similarly, derived objects can take arguments that are supertypes of the defined arguments of the virtual function. This is known as contravariance. 

```C++
    // in Vehicle:
    virtual void fix(const Car & c) {/*...*/}

    //in Car:
    void fix(const Vehicle & v) override {/* .. */}
```



