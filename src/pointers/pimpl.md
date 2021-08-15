# PIMPL

PIMPL stands for pointer to implementation and is also known as a compiler firewall. Consider the following header file:

```C++
#include <ie_core.hpp>
#include <string>
#include <vector>
using namespace InferenceEngine;
class NeuralNetwork {
    Core ieCore;
    ExecutableNeuralNetwork net;
    std::string modelPath;
public:
    SemanticSegmentationNetwork(const char * model, const char * device = "CPU");
    std::vector<float> infer(const std::vector<float> & input);
};
```
Firstly, every client of this code is going to end up including `string`, `vector` and OpenVino's `ie_core.h`. 
Since `std::string` and `std::vector` are part of the standard library, they're not really an issue. 
But clients shouldn't know or care about the implementation of `NeuralNetwork` and they shouldn't need to depend on the `ie_core.hpp` header file since they never directly use it. 
However, the above implementation will include headers from OpenVino for anything that uses a `NeuralNetwork`. 
Now, if we decide to update our version of OpenVino and there is a change to `ie_core.hpp` (which may also include other header files) we'll have to recompile everything that uses the `NeuralNetwork` 
interface even though the changes to the library only affect the implementation of `NeuralNetwork`.

We can ameliorate some of these issues by forward declaring the OpenVino classes we use. 
Instantiating a template or an object of a class requires that class's definition. 
But declaring the signature of functions that accept/return a forward declared type (known as an incomplete type) or declaring (but not initializing) variables of a forward declared type is legal.

```C++
#include <vector>
#include <string>
// no include for ie_core.h
namespace InferenceEngine {
    class Core;
    class ExecutableNeuralNetwork;
}

class NeuralNetwork {
    InferenceEngine::Core ieCore;
    InferenceEngine::ExecutableNeuralNetwork net;
    std::string modelPath;
    class MyClass myClass; // can forward declare type and declare member in one line too
public:
    SemanticSegmentationNetwork(const char * model, const char * device = "CPU");
    ~SemanticSegmentationNetwork();
    std::vector<float> infer(const std::vector<float> & input);

};
```

With this method, we no longer are able to define the default constructors and destructors in the header file. 
This is because at the time the class is defined, `Core` and `ExecutableNeuralNetwork` don't have definitions. 
So the compiler has no idea how exactly to destroy or create these members. 
Instead, we must declare the destructor in the header file, and define it separately in the source file after the compiler has definitions for all the class's members.

This is an improvement, but clients still have visibility of `NeuralNetwork`'s internals. 
While access modifiers prevent client code from using internal members, it can do nothing to stop clients from knowing it exists. 
Private members participate in name lookup even if they are encapsulated away. 
Furthermore, a client using this class that comes to the header file to read documentation is fully exposed to all private and protected data members. 
They could also see any implementor comments that correspond to internal members.

We can create a compiler firewall and improve encapsulation with the pimpl idiom (**p**ointer to **impl**ementation).

In the header file:
```C++
#include <memory>
#include <vector>
class NeuralNetwork {
    struct Impl; // declaration for Impl
    std::unique_ptr<Impl> pimpl;
public:
    NeuralNetwork(const char * model, const char * device = "CPU");
    ~NeuralNetwork();
    std::vector<float> infer(const std::vector<float> & input);

    // forward declaration in function
    class MyClass doIt(const class MyClass & cl);
};
```

In the source file:
```C++
#include "NeuralNetwork.h"
#include <string>
#include <ie_core.hpp>
using namespace InferenceEngine;
class MyClass {}

struct NeuralNetwork::Impl {
    Core core;
    ExecutableNeuralNetwork net;
    std::string modelPath;
    MyClass m

    Impl(const char * path) : modelPath(path) {}
};

NeuralNetwork::NeuralNetwork(const char * model, const char * device) : pimpl(std::make_unique<Impl>(model)) {}
// Note: do not redefine default arguments

~NeuralNetwork::NeuralNetwork() = default;
// need to define the dtor after we create a definition for Impl
// since the compiler needs to know how to destroy it

std::vector<float> NeuralNetwork::infer(const std::vector<float> & input) {
    // ...
}

MyClass NeuralNetwork::doIt(const MyClass &) {
    //..
}
```

C has "true" encapsulation.
Foo.h
```C

struct Foo * foo_make();

int foo_getSomething(struct Foo *);

void foo_doSomething(struct Foo *, int);

void foo_free(Foo *);
```

Foo.c
```C++
struct Foo {
    int bar;
};

struct Foo * foo_make() {
    Foo * ptr = (Foo*)malloc(sizeof(Foo));
    ptr->bar = 0;
    return ptr;
}

int foo_getSomething(struct Foo * foo) {
    return foo->bar;
}

void foo_doSomething(struct Foo * foo, int a) {
    foo->bar = a;
}

void foo_free(Foo * ptr) {
    free(ptr);
}
```

The internals of `Foo` are **completely** encapsulated away. 
A client has no idea what `Foo` is, they just see they can make and free it, along with `doSomething` and `getSomething`. 
A client is protected from whatever header files `Foo` needs because the implementation is isolated in its own file.

C++ classes cannot do this because a compiler needs to know their size. 
Pimpl gives us a way to achieve this by making the actual class relatively small and just contain a pointer to a struct that contains all the internal members.

Of course, pimpl does incur the cost of the indirection and the heap allocation, and probably isn't necessary in every class you develop. 
However, with that being said, it's a great idiom to provide increased encapsulation and isolation. 
Private member variables and functions could be left out of the header file, leaving the header easy to read for clients to get information on the public interface of the class.

# Smart Pointer Polymorphism

It works like they were regular pointers or references. However, covariant return types only work with pointers and references. 
Furthermore, `std::unique_ptr<T>` and `std::unique_ptr<const T>` are two completely different instantiations of the template class and therefore not implicitly convertible. 
On top of that, while the type `D` may subtype `B`, it is **not** true that `smart_ptr<D>` subtypes `smart_ptr<B>` 
even though certain function such as constructors permit using a smart pointer or regular pointer to a derived type.

```C++
class Base {
public:
    virtual std::unique_ptr<Base> clone() = 0;
    virtual int getInt() const = 0;
    virtual ~Base() = default;
};

class Derived : public Base {
public:
    std::unique_ptr<Base> clone() override {
        return std::make_unique<Derived>();
    }

    int getInt() const override {
        return 10;
    }
};

std::unique_ptr<Base> useBase(std::unique_ptr<Base> && b) {
    std::cout << b->getInt() << std::endl;
    return b;
}

std::unique_ptr<Base> obj = std::make_unique<Derived>();

obj = useBase(std::move(obj));
```

The only part of a smart pointer that can act polymorphically is its constructor, assignment operator, and swap member function. So if type `D` subtypes `B`, 
we can construct a `smart_ptr<B>` from a `smart_ptr<D>` or `D*`.

```C++
void funcA(std::unique_ptr<B> &) {}
void funcB(std::unique_ptr<B> &&) {}

void funcC(std::unique_ptr<const D> &) {}

auto d_ptr = std::make_unique<D>();

funcA(d_ptr); // error!, std::unique_ptr<D> is not a subtype of std::unique_ptr<B>
funcB(std::move(d_ptr)); // fine, constructs a std::unique_ptr<B> from std::unique_ptr<D>
// since D subtypes B

funcC(d_ptr); // error! std::unique_ptr<D> is not std::unique_ptr<const D>
```

For polymorphic function arguments, it's best to just accept references. This allows automatic variables, smart pointers, and raw pointers to use the function. 
It also allows implicit conversion to `const`.

```C++
void funcD(const B &) {}

auto d_ptr2 = std::make_unique<D>();

funcD(*d_ptr2); // good
// D& is convertible to const B &
```