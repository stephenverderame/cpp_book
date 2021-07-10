# Inheritance

We've seen public inheritance in the last chapter. It has the syntax `: public Base` and models an "is-a"/"subtype-of" relationship. Before going further, let's look at a classic example of where the wording "is-a" can lead us astray. Is a square a rectangle? Mathematically, yes. But in computer science, that depends...

Consider:
```C++
class Rectangle {
protected:
    int length, width;
public:
    Rectangle(int l, int w) : length(l), width(w) {}
    virtual ~Rectangle() = default;

    virtual void setLength(int l) {
        length = l;
    }

    virtual void setWidth(int w) {
        width = w;
    }

    int area() const {
        return length * width;
    }

    int perimeter() const {
        return 2 * length + 2 * width;
    }

    int getWidth() const { return width; }
    int getLength() const { return length; }
};

class Square : public Rectangle {
public:

    Square(int sideLen) : Rectangle(sideLen, sideLen) {}

    void setLength(int l) override {
        length = width = l;
    }

    void setWidth(int w) override {
        length = width = w;
    }
};


Square s(5);

Rectangle & r = s;
// ...

// Now image we don't know that r is a square
// which is the entire point of Polymorphism
r.setLength(10);
// image the surprise when we find that the width changes as well!
r.getWidth();

```
In this example, the interface of Rectangle doesn't match Square. If we wrote specs for Rectangle's `setLength` and `setWidth`, the likely specification "sets the Rectangle's length/width" clearly doesn't match the behavior of setting both the length and width! While a square *is a* rectangle by definition, it does not adhere to the interface. Based on this code snipped, we are inheriting from Rectangle for code reuse. A much better alternative would be to make `area()` and `perimeter()` free functions. In this simplistic case, we can make `Square` and `Rectangle` separate `structs` with their internals known.

```C++
struct Square { 
    int sideLen;
};

struct Rectangle {
    int length, width;
};

int area(int length, int width) {
    //...
}

int perimeter(int length, int width) {
    //...
}

// or if you'd prefer:

int area(const Square & s) { 
    return area(s.sideLen, s.sideLen); 
}

int area(const Rectangle & r) {
    return area(r.length, r.width);
}

```


Now I'd like to distinguish between implementation and interface inheritance. Interface inheritance is when a class inherits from an interface (pure virtual functions) to implement it  and provide decoupling between interface and implementation(s). Implementation inheritance is inheritance in order to share code (square and rect example). We want to be wary of implementation inheritance. Inheritance is a very strong coupling relationship, and therefore implementation inheritance should be avoided when possible. Here's an example of interface inheritance:

```C++
class Port {
public:
    /**
    * Writes the entire buffer to the port
    * @throw std::runtime_exception if the write failed
    */
    virtual void write_all(const std::vector<std::byte> & data) = 0;

    /**
    * Blocks until it reads the specified amount of bytes
    * @param size the amount of bytes to wait for
    * @throw std::runtime_exception on fail
    */
    virtual std::vector<std::byte> read_all(size_t size) = 0;

    /**
    * Reads data currently available on the port
    * May or may not read all the data available
    * @return the data read from the port or an empty vector if nothing
    *   is available
    */
    virtual std::vector<std::byte> read_nonblock() = 0;


    /**
    * Gets the amount of bytes <= to the amount of bytes available to be read
    * Guaranteed to return at least 1 if there is any data available
    */
    virtual size_t available() const noexcept = 0;

    virtual ~Port() = default;
};

class MemoryPort : public Port {
    // Implement methods for reading/writing to an area in memory
    // useful for testing
};

class Socket : public Port {
    // Implement interface for reading/writing to a socket
};

class Serial : public Port {
    // Read/write to a serial port
};

class FDPort : public Port {
    // Read/write to a pipe or file
};

void logError(Port & port, const std::string & str) {
    // do some logging
    
    // log doesn't know or care the implementation of Port
    // Are we logging to the console? a file? logging to a pipe to another
    // program which will respond to the errors?
    // are we a remote service logging to another service via sockets?
    // are we a slave logging to the master over a serial connection?
    // don't know and don't care!
}

```

All of these types are subtypes of `Port`. They can be used polymorphically and prevent users from depending on or knowing about any implementation. This is the hallmark of OOP and is known as *dependency inversion*, which we'll discuss later.

With all that being said, let's look at a tool specifically designed for implementation inheritance: private inheritance. Private inheritance is same as public inheritance, but all public members of the base class become private members in the derived class. This models an "implemented-in-terms-of" or a "has-a" relationship which is the same relationship modelled by object composition. Therefore, you should prefer composition to private inheritance. Truthfully, I can't remember a time when I've used private inheritance.

Private inheritance is not polymorphic. This intuitively makes sense since every part of the Base class's interface is private. If it were polymorphic, then you could break encapsulation by changing the static type of the Derived class to the Base class and call the Base class member functions.

Syntactically, the only difference is `: private Base` instead of `: public Base`.

## Multiple Inheritance

C++ allows inheriting from multiple base classes. This should really only be used for interface inheritance.

```C++
class Drawable {
public:
    virtual void draw() = 0;
};

class Entity {
public:
    virtual bool isAggro() = 0;
    virtual void takeTurn() = 0;
};

class Player : public Drawable, public Entity {
    // multiple interface inheritance


    // implement functions
};

```
 
If used for interfaces only, multiple inheritance is straightforward. But consider the following:
```C++
struct A {
 protected:
    int aVar;
    void doA();
 }
 class B : public A {}
 class C : public A {};
 class D : public B, public C {};
```
 This diamond shaped hierarchy is best avoided, but if it does happen it might not be clear how it will behave. Remember, derived classes and their members are simply tacked on to the base class. Therefore, here we inherit from `A` twice and get two copies of the variable `aVar`. To use it, we must explicitly qualify which parent's `aVar` to use.
```C++
 D d;
 d.B::aVar = 0;
 d.C::aVar = 1;
 // two copies of the same variable
 
 // or from within D:
 void D::doStuff() {
    int a1 = B::aVar;
    int a2 = C::aVar;
 }
```
 
 The way to avoid this is *virtual inheritance*. Virtual inheritance enforces that a base is only inherited once. Implementations vary but this generally could be implemented by derived classes holding pointers to their parent classes. Virtual inheritance is beefier and more expensive.
```C++
 class A {};
 class B : public virtual A {};
 class C : public virtual A {};
 class D : public B, public C {};
```

 It's good to know these features exist. But the best method for dealing with these problems is to not create them in the first place. Multiple inheritance should be used mainly for representing subtyping of multiple distinct interfaces.

 ## Final

 You can use the `final` keyword to declare a function to be the final overrider. Subclasses of the class where the method was declared `final` will no longer be able to override it. `final` should be used sparingly.

 ```C++
class Base {
public:
    virtual void doIt() = 0;
};

class Derived1 : public Base {
private:
    virtual void doA() = 0;
    virtual void doB() = 0;
public:
    void doIt() final {
        doA();
        doB();
    }
};

class Derived2 : public Derived1 {
    void doA() override { /*... */}
    void doB() override {/*...*/}

public:
    void doIt() override {} //error!
}
 ```

 It's best to only have 1 of `final`, `virtual`, or `override`. For the declaration of a virtual function, use `virtual`. Then all derived classes should use `override` or `final` (and not both).

 ## Default Arguments

 Default arguments of methods are determined by the static type of the context object. To avoid problems, do not override the default arguments present in virtual functions of the base class.

 ```C++
class Logger {
public:
    virtual void log(const std::string & msg = "Hello") = 0;
};

class ErrorLogger : public Logger {
public:
    void log(const std::string & msg = "Error!") override {
        // bad practice, should not override default arguments
        //...
    }
};

ErrorLogger log;
log.log(); // default argument "Error!"

Logger & l2 = log;
l2.log(); // default argument "Hello"

 ```

 ## Alternatives to Inheritance

* Object composition and delegation
    * Object's members are other classes which it delegates some responsibility to by calling its methods
* Interfaces
    * Subtyping
* Mixins
    * Think of creating a set of related, reusable functions, naming this set, and extending a class with it
    * In C++ this can be realized by inheriting from abstract classes that **do not** have `protected` data members.