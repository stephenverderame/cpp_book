# Type Erasure

What if we needed to store a vector of different types? Well, if we wrote all the classes we want to store in the vector, we could use inheritance and store `std::reference_wrapper`s to the base class.
Instead, let's say we have a bunch of different classes with a similar static interface, but are not part of an OOP hierarchy. For that, we need type erasure.

The gist of type erasure is to create a template holder class which does nothing but forward function calls to object it contains. Then we make this holder class inherit from our concept interface.
This inheritance is key to allow multiple different types (instantiations of the holder, aka model, class) to be used where an interface is expected. We can then store `reference_wrapper`s of this
Concept interface in the vector, or wrap the concept in our own class. That probably made no sense, so let's break it down.

Let's say we have these two concrete classes:
```C++
struct Duck {
    void quack() {};
};

struct Duckish {
    void quack() {};
};
```

The concept we are trying to store in our container is something that sounds like a duck (has a `void quack()` method). 
Since these two classes are fundamentally different types, we cannot store them together in a container.

```C++
struct DuckLike {
    virtual ~DuckLike() = default;
    virtual void quack() = 0;
};
```

What we would like to do is something like this:
```C++
std::vector<std::reference_wrapper<DuckLike>> ducks;
```

However, our concrete classes (`Duck` and `Duckish`) do not implement our concept interface (`DuckLike`). So we need an intermediary step. 
We'll create a template that requires all template type arguments to uphold the static interface of our concept. Then, we'll make this template implement the dynamic interface.
Effectively, what type erasure is doing is converting static polymorphism into dynamic polymorphism, or said another way: generic polymorphism into oop polymorphism.

```C++
template<typename T>
class DuckHolder : public DuckLike {
    T duck;
public:
    template<typename U>
    DuckHolder(U&& duck) : duck(std::forward<U>(duck)) {}
    // a universal reference must be on a template function
    // if we used T, the constructor wouldn't be a template and T&& would
    // be an rvalue reference

    // this is because there would only be one constructor for DuckHolder<T>
    // a universal reference accepting function, essentially becomes multiple
    // in the compiled binary (like any other template) so that you can bind
    // both rvalue and lvalue references to it

    void quack() override {
        duck.quack();
    }
}
```

Using our DuckHolder, we can then use generic polymorphism where dynamic polymorphism is required.

```C++
ducks.push_back(DuckHolder{ Duck() });
ducks.push_back(DuckHolder{ Duckish() });
```

Instead of using `std::reference_wrapper`, we could also define our own wrapper object, or use smart pointers.
