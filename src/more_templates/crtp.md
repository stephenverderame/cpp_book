# Curiously Recurring Template Pattern

CRTP is a template pattern where a derived class inherits from a template base class, and the derived class is the type argument for the base class.
So something like this:

```C++
template<typename T>
class Base {
    Base() = default;
    friend class Derived; // allow Derived to construct Base
    // This private constructor makes the Base only constructable
    // from the Derived class. This prevents us from using Base incorrectly

    // static_asserts or SFINAE could also be used

};

class Derived : public Base<Derived> {

};
```

So what's the purpose of this? Well with this setup, code in the Base class automatically knows what the dynamic type is because the actual type is passed as a template argument.
This makes it safe to `static_cast` instead of `dynamic_cast` which saves the runtime cost of RTTI type checks. This ability can be used to implement a generic version of
mixins. 

In OOP, a mixin is basically a small class that provides a named set of functionality to different types. These mixins are not designed to stand on their own and are typically narrow in scope.
For example, you can have a mixin that might provide serialization functionality. Instead of each class having a `to_json` and `from_json` method, you can have all classes that you might want
JSON serialization functionality for inherit `JSONSerializationMixin` which provides said methods. 
As you can imagine, there isn't necessarily much in common between the "children" of a mixin; it doesn't define a hierarchy.
You can have a `Device` class use the mixin, and a `Student` class; there isn't necessarily a "sibling" relationship.

So back to CRTP, we can use it as a way to add functionality, similar to a mixin, that can be shared between different classes.

```C++
template<typename T>
class Geometry {
    Geometry() = default;
    friend class Square;
    friend class Rectangle;

    int geom_length() {
        if constexpr (std::is_same_v<T, Square>) {
            return static_cast<T&>(*this).sideLen();
            // static cast is safe
        } else {
            return static_cast<T&>(*this).length();
        }
    }

    int geom_width() {
        if constexpr (std::is_same_v<T, Square>) {
            return static_cast<T&>(*this).sideLen();
        } else {
            return static_cast<T&>(*this).width();
        }
    }

public:

    int area() {
        return geom_length() * geom_width();
    }

    int perimeter() {
        return 2 * geom_length() + 2 * geom_width();
    }

    int volume(int height) {
        return area() * height;
    }
};

class Square : public Geometry<Square> {
puiblic:
    int sideLen() { /*...*/ }
};

class Rectangle : public Geometry<Rectangle> {
puiblic:
    int length() { /*...*/ }
    int width() { /*...*/ }
};
```

`static_cast` won't remove `const`, so if calling a constant member you must cast to a `const` reference to the derived type.

```
//Usage:

Square s;
s.area();

template<typename T>
void doFoo(Geometry<T> & geom) {
    // being a type argument to Geometry already limits what T can be
    // no need for SFINAE or even a requires clause in the spec

    std::cout << geom.volume(10) << std::endl;
}

Rectangle r;

doFoo(r);
doFoo(s);

```

A benefit of this over template non-member functions is that this makes it very clear which classes support which helper functions. 
Consider that we also had a `Circle` class. As written the functions in `Geometry` won't support our `Circle`, so if `Geometry` was a set of non-member functions,
we'd have to document that they didn't support `Circle` and we could also use SFINAE to ensure the user doesn't accidentally pass a `Circle`.
And look, I'm basically a crazy masochist and even I get tired of writing out all the structs to emulate C++20 concepts when I can't use them.
Furthermore, it's an added burden on the user to have to remember what helper functions use which classes. CRTP also makes these non-member functions clearly part of the interface of Square and Rectangle.
This isn't to say that CRTP is the best solution, but it's an alternative.

I view common applications of CRTP and SFINAE as *defensive programming* techniques. The goal here is to make your interfaces idiot proof.
Imagine there is a little gremlin (maybe you at 2AM) who is trying to use your interfaces incorrectly; template non-member functions that bind to any type
are good targets for this nasty little guy. Both SFINAE and CRTP can be applied to restrict what types can be passed to these functions, thus making them
more protected from misuse.

Notice that CRTP doesn't use any virtual functions. While this does cut down on runtime cost, it also makes functions in the base class susceptible to shadowing.
If you use this pattern, you should ensure that the base class members have different names than any member in any derived class.

So that's one application of CRTP. Another one is to create static interfaces similar to how the STL container adapters work. 
The advantage with CRTP over an adapter is that we can use private members. Let's create an interface that provides a `size` method.

```C++
template<class T>
class Container {
    Container() = default;
    friend class MyRange;
    friend class MyVec;

public:
    size_t size() const {
        return static_cast<const T&>(*this).size();
    }
};

class MyRange : public Container<MyRange> {
public:
    size_t size() const {
        return std::distance(begin, end);
        // imagine this class stores a begin and end iterator
    }
};

class MyVec : public Container<MyVec> {
public:
    size_t size() const {
        return dataSize;
        // dataSize could be an internal member storing size
    }
};
```

Notice the subtle difference in flow of control. In this example, the implementation is in the derived class while in the previous one the
implementation was in the base class.

The name collision in this case is safe since the base and derived class `size` member functions do the same thing.

Now to use this:
```C++
MyRange mr;
MyVec mv;

template<typename T>
void doFoo(const Container<T> & container) {
    std::cout << container.size() << std::endl;
}


doFoo(mr);
doFoo(mv);
```