# Policy Based Design

I've talked about this a bit before in the main templates chapter. This is a continuation of that discussion.

In a way, PBD is another form of generic mixins. We design small, narrow focused classes, that provide functionality to a wide range of unrelated classes. 
A big difference is that PBD focuses on providing points of customization. Not only are we extracting out repeated functionality into its own class, but by
making these policies a template argument, we allow users to easily change behavior of a policy-holder class by changing one of its policies.

We've seen this with allocators, smart pointer deleters, `std::char_traits`, comparators, predicates, and hash functions in the STL.

## Template-template Parameters

A common technique used with PBD are *template-template parameters*; template type parameters that are themselves template types. This is especially useful
when a type passed directly to the policy-holder would also be passed to its policies as well.
```C++
// Template Arguments:
MyVec<int, MyCmp<int>, MyAllocator<int>, MyHash<int>>

// Template-Template arguments
MyVec<int, MyCmp, MyAllocator, MyHash>
```

The syntax for such a type argument is to put a `template<typename>` before the `typename` of the policy in the template argument list:
```C++
template<typename T,
    template<typename> class Comparator,
    template<typename> class Alloc = MyAllocator,
    template<typename S> class Hash = MyHash> //name this template-template argument as S
```
`template-template` parameters can have multiple template arguments, they could also be given template argument names. When you use these arguments as concrete types, you must also specify the type parameter.

```C++
// template arguments above
class MyVec {
    T * data;
    size_t data_sz;
public:
    explicit MyVec(size_t len) :
        data(MyAllocator<T>::alloc(len)) {}
        // MyAllocator is a template, we must pass the type to it
        // call static function alloc on MyAllocator
    
    ~MyVec() {
        MyAllocator<T>::free(data);
    }
};
```

## Dependent Names

A dependent name is essentially a name that relies on a template argument to get resolved. Basically these are the members of a template type or template type argument.
By default, the compiler interprets dependent names as values, which is why we needed the `typename` keyword when accessing a member type alias.

```C++
template<typename T>
using type = MyVec<T>::type;
// Compiler has no idea what `type` is
// by default it assumes it's some value
// which is why we need to add `typename`
```

The compiler could wait until the template is instantiated before checking this, however many compilers don't in order to give the error to the author of the code and not an unsuspecting client.

Consider the following:
```C++
T::doSomething<int()>(5);
```
The compiler could interpret this three ways:
1. Call a function `doSomething` that takes a type argument which is a callable object that accepts no parameters and returns `int` and pass that function `5`
2. Instantiate a template type `T::doSomething` with the template argument `int()` (type of parameter-less callable object returning `int`) and initialize it with `5`.
3. Call `operator<` on a value `T::doSomething` with the no-parameter initializer for type `int` (which is 0) and then check if the integral value of the resulting boolean is greater than 5.

We can tell the compiler to use option 2 with the `typename` keyword, and option 1 with the `template` keyword.
```C++
T::template doSomething<int()>(5); // #1
typename T::doSomething<int()>{5}; // #2
T::doSomething < int() > (5); // #3
```

Generally speaking, use `typename` to access a type alias of a template type, and use `template` to call a template method of a template type. `template` can also be used after `.` and `->` too.

---
For more information, take a look at [Modern C++ Design](http://index-of.co.uk/C++/C++%20Design%20Generic%20Programming%20and%20Design%20Patterns%20Applied.pdf) by Andrei Alexandrescu.