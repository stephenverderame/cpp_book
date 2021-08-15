# OOP and Class Guidelines

## Prefer Non-member non-friend functions

A non-member, non-friend function can only use the public interface of a class. 
This promotes encapsulation by only allowing the function to access data it needs. 
Of course, there are plenty of functions that need to be a member or access private data. 
These types of functions can be kept as members, but convenience functions and other operations that can be implemented with the public interface should be non-members so long as they don't incur 
*premature pessimization*. This helps promote the *coherence* of a class. Coherence is basically a measure of how many member variables are used in a member function. 
Ideally, every method of a class uses every member variable. Poor coherence indicates that the class may be doing too many things and therefore exposing members to functions which are unrelated.

```C++
class RingBuffer {
public:
    size_t size() { /* ... */}
};

bool empty(const RingBuffer & rb) {
    return rb.size() == 0;
}
```

There's actually a TS (technical specification, basically a proposal for a language or library feature) for *unified function calls* which would allow `empty()` 
to be called as if it were a member function or `size()` to be called as if it were a free function. Will this ever be part of a future version of C++? Not sure.

## NVI Idiom

NVI stands for non-virtual interface, and it's basically an application of GOF's Strategy Pattern. 
The idea is to have a public interface which is non-virtual (public virtual destructor not included) and provide private or protected methods which serve as hooks of customization by subtypes. 
This allows breaking up a complex computation into smaller customizable steps and/or enforcing pre and post conditions which all subtypes must uphold in the supertype.

A private virtual function can be overriden by subtypes, but unlike protected functions, the subtype cannot call it directly.

```C++
class MsgFormatter {
    virtual std::vector<std::byte> _format(const std::vector<std::byte> & data) = 0;
    virtual std::vector<std::byte> str_to_bytes(const std::string & str) = 0; 
public:
    void std::vector<std::byte> format(const std::string & data) {
        // assert preconditions
        auto res = _format(str_to_bytes(data));
        // assert postconditions
        return res;
    }

    virtual ~MsgFormatter() = default;
};

class AsciiMsgFormatter : public MsgFormatter {
    //..
    // override _format and str_to_bytes
};
```

In this example, if `_format` were independent of `str_to_bytes` it might be better to use *mixins* or *policy based design* (we'll talk about this later) but that's kind of the idea.

## Other Guidelines

* [C.133](http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rh-protected) - Avoid `protected` data. This is a sign of implementation inheritance and, as shown, can be quite annoying when used with multiple inheritance.
* [C.164](http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Ro-conversion) - Avoid implicit conversions
* [C.131](http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rh-get) - Classes should generally be told to do something, not get at their internals. If a class has quite a few trivial getters and setters, then that could be a sign that the internals should be public and a `struct` might be a better choice.
* [C.12](http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rc-constref) - Class members probably shouldn't be `const` or references. These types of members can only be set once. So we can construct object with these types of members by setting them in the constructor, but we can't copy them using `operator=`.
* [C.160](http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Ro-conventional) - Don't be cute with overloads. Overload operators for operations that make sense in C++. 