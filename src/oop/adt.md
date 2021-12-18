# Interfaces

What we saw last chapter was a *concrete* class. Concrete types have their representation part of their definition. 
Unlike concrete types, abstract types cannot be instantiated, but instead provide an interface for other concrete classes to implement. 
Abstract types must be used via pointers or references, while concrete types can be used directly. Concrete types can be placed on the stack, and be members of other classes much more simply than abstract types.

In Java and other languages, all functions of a superclass may be overridden. This isn't the case in C++. A function that you want a derived class to be able to override must be declared `virtual`. 
The reason is that the presence of a `virtual` function requires something known as a virtual table or vtable. This is an area in memory that each instance has access to that allows the 
compiler to perform dynamic dispatch by looking up the specific function to call at runtime. More accurately, each instance has a pointer to its respective virtual table. 
Since C++ is "pay for what you use", this vtable isn't created unless a class declares a function `virtual` which enables dynamic dispatch for that function and allows subclasses to override it.

An abstract type contains at least one *pure virtual* functions, which is a function that subclasses **must** override because the superclass does not implement it. 
The C++, the equivalent of a Java `interface` would be a class containing only pure virtual functions.

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

When overriding a function, it's good practice to explicitly denote it as such with `override`. 
This prevents you from accidentally creating a new function and *shadowing* the super class's functions. 
Since you cannot overload superclass functions, creating a function with the same name as a superclass's function in a subclass shadows that function and prevents it from being called.

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

Inheritance enables us to use polymorphism and let subclasses behave like their superclasses. 
A user would not need to care about the concrete class or its implementation, they can just use the interface defined in the supertype without regard to the specific implementation. 
However, in order to fully support this, we need to ensure that the destructor is virtual. 
Otherwise, there will be no dynamic dispatch on the destructor which will cause undefined behavior if we destroy a subclass through a reference or pointer to the superclass. 
Another alternative is to make the base class destructor protected and non-virtual. This will prevent deletion of a derived type through a pointer or reference to the base class.

Constructors on the other hand, cannot be virtual. This is because construction requires complete type information; the static type is the type being constructed. 
Furthermore, the class doesn't exist as an object at runtime yet, so you can't call a virtual method on it.

As an addendum, you should not call virtual functions in constructors or destructors. 
This is because the actual type of the object changes during these two operations, and the actual type is always the type being created/destroyed. 
During construction of a derived class, we first start by constructing the base class and running the base class constructor. 
In this constructor, the actual type of the object is the base class type and not yet the derived class type. 
Then we build off the base and construct the derived class by calling the derived class constructor. 
During this second constructor call, the actual type changes to be that of the derived type. 
For destruction, the process is similar but in reverse, destroying the derived object before destroying the base.
Therefore, if you call a virtual function in the base class constructor, it will dispatch to the implementation in the base class and not the derived class. This is because, when the base class constructor is being run, the dynamic type is still the base type.
So calling virtual functions in constructors/destructors is technically safe so long as the virtual function is not pure virtual and you don't expect it to dispatch to a derived type, but it's not a good idea.

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

`Vehicle` has a constructor, but it is an abstract type so it still can't be constructed directly. 
Instead, subclass constructors must delegate one of the superclass constructors like shown. 
When delegating a call from one constructor to another, we cannot use the initializer list to instantiate extra variables.

Also, notice how `moveIt` takes a `Vehicle` by reference. 
This is paramount to avoid *object slicing*. In memory, a concrete derived class is formed by taking its state and tacking it on to the data of the superclass. 

|&Car|
|---|
|Vehicle Data|
|`std::string name`|
|Car Data|
|`int horsePower`|

Passing the derived class by value as a superclass instance will only copy the data that the subclass shares with the superclass. If the derived class adds additional data members, then these data members won't be copied because the compiler thinks its just dealing with an instance of the superclass. Hence the name object slicing, because the data specific to the subclass is sliced off. Another piece of information which is sliced off the the virtual table pointer of the base class. Consider the following:

```C++
class Base {
public:
	virtual ~Base() = default;

	virtual void speak() {
		printf("Hello\n");
	}
};

class Derived : public Base {
public:
	void speak() override {
		printf("Derived\n");
	}
};

void slice(Base b) {
	b.speak();
}

Derived d;
slice(d);
```

What we'll end up with is `Hello` being printed to the console. When we copy an object like this, behind the scenes the compiler invokes the copy constructor, which is not virtual. Moreover, copies don't copy the virtual table pointer. Why? Well, suppose that it did. Then invoking virtual methods like `speak()` would dispatch to the derived type. But the derived type implementation might use data members that are not shared between the base and derived class. Since we already discussed that these members could not be copied over, then such a function invocation would give us undefined behavior by accessing invalid memory.
Therefore, object slicing prevents dynamic dispatch from operating as expected. 

In the `Vehicle` example, if `Vehicle` was passed by value, the compiler would complain since that would require it to construct a new instance of `Vehicle`, and `Vehicle` is abstract and cannot be constructed.

Now even with references we can run into a bit of a conundrum:

```C++
Car c1(300), c2("car 2", 500);
Vehicle& v = c1; //reference v being bound to c1, good
v = c2; //operator=, uh oh
v.move(); // ?
```

As we saw in the last chapter, `operator=` is not normally virtual. We can make it virtual and overload it ourselves, but by default it's not. 
Therefore, the above example causes object slicing as well! This is because `operator=` gets called on the static (declared) type which is `Vehicle` and not the dynamic (actual) type which is `Car`. 
So here, `operator=` will copy only the members it knows about (the ones that are part of the static type) and leave the rest unchanged. So the output of `v.move()` is "car 2 went 30 mph". And since `v` is basically an alias for `c1`, that's the same output for `c1.move()`. So we see here that we "half-copied" `c2` to `c1`!

## Covariance and Contravariance

Suppose we had a function `clone` in our previous hierarchy. In the interface, we might define it as follows:

```C++
    virtual Vehicle* clone() = 0;
```

But in `Car`, we know that if we clone a `Car` we'll get another `Car` back, so it would sure be nice to override it like such:

```C++
    Car* clone() override {/* ... */ }
```
Well we actually can! This is known as *covariant* return types and it permits derived classes to return an object that is derived from the return type of the virtual function. 
This is not an overload, and in fact, two functions that differ in **only** their return types are not overloads.

Similarly, derived objects can take arguments that are supertypes of the defined arguments of the virtual function. This is known as contravariance. 

```C++
    // in Vehicle:
    virtual void fix(const Car & c) {/*...*/}

    //in Car:
    void fix(const Vehicle & v) override {/* .. */}
```

### Possible Exercises

1. Create a `Logger` interface and at least two concrete subtypes. One for logging to the console and one for logging to a file. (`std::fstream` may help out). Also create a `LogLevel` enum that allows differentiating the severity of the message between at least 3 severity levels. The `LogLevel` should change the display of the log in their respective medium. Perhaps for the console logger you can change the color with ANSI escape codes and for the file logger use textual features such as capitals or markdown symbols like underscores and asterisks. The interface should have at least 1 function, which could take a string message and log level. Try using the `Logger` polymorphically.

