# Pointers

I've been skirting around this topic for a bit because I wanted to start with the modern C++ smart pointers, and then mention the C style raw pointers. 
However, I think understanding the smart pointers takes some groundwork, which I think we finally have laid out.

The C++ smart pointers are template types, that is to say they are classes which take template type arguments. Like raw pointers and references, smart pointers enable the class they wrap to behave polymorphically. 
The three smart pointers (`unique_ptr`, `shared_ptr`, and `weak_ptr`) all have slightly different semantics and uses, but taken as a whole they have a few key advantages over "dumb" pointers:

1. Firstly, smart pointers are RAII classes for regular pointers. The smart pointer itself has an automatic lifetime, so when it goes out of scope it handles any resource management it may need to do. 
This is a huge benefit and motivator for smart pointers, and already enough reason to prefer smart pointers to normal pointers.
2. Smart pointers make the relationship between the pointer and data clear. 
For example, a raw pointer may be something you cannot delete such as an address to data managed by another object or an address of an automatic object like a static or local variable. 
It can also be something you must delete such as a resource the pointer owns. On top of that, for each of these categories, a pointer can be a pointer to a single piece of data or an array. 
And if it does point to an array there's no way of knowing the amount of elements in that array from just the pointer alone. Put simply, without comments or more context, raw pointer are ambiguous.
3. Smart pointers prevent heap mismatches. On top of remembering to delete the allocated memory, you must remember to match the correct method of deletion to the way it was allocated. 
`new` pairs with `delete`, `new[]` pairs with `delete[]`, and `malloc()` or `calloc()` pairs with `free()`. 
AND, the component that performed the allocation should be the component to perform the deallocation as well. Let's say you use a library that allocated data with `new` and gives you back a pointer to that data. 
Although `delete` is the matching de-allocation function, you shouldn't call it directly! Instead, you should call a delete method provided by the library. 
The reason is that the library may have been built with a different STL implementation that does different housekeeping for resource management, the library allocation may be occurring on a different heap, 
or the library may do house-keeping of its own that assumes you call its own resource cleanup function. 
These are the types of bugs that may not show up for a while and then suddenly rear their head after you make some completely unrelated change.
4. Less time developing your own RAII. You yourself may need a custom allocator or extra logic for allocations and de-allocations. 
Instead of writing your own RAII, smart pointers provide an out-of-the box and tested solution.

## Basic Usage

Smart pointers belong in the memory header which is included with `#include <memory>`.

Smart pointers overload `operator*` and `operator->` in order to be used similarly to a raw pointer. 
As a reminder, `operator*` is the dereference operator and it returns a reference to the underlying data. `operator->` is like the dot operator, but for pointers. 
It gets members that belong to the underlying data. `operator->` is repeatedly applied until it gets to an object that does not overload it. 
So calling `operator->` on the smart pointer gets you right at the member functions of the internal pointer instead of having to do something like `->->`. 
In truth, `operator->` is implemented the same way as `operator*` (by returning a reference to the underlying data), however they differ semantically.

Smart pointers also are comparable with `nullptr` and convertible to `bool`. A smart pointer that evaluates to `true` has non-null data while one that evaluates to `false` is essentially a `nullptr`.

```C++
struct MyStruct {
    int a, b, c;
};

std::unique_ptr<MyStruct> structPtr = new MyStruct(10, 20, 30);
structPtr->a; //10
structPtr->b = 50;
MyStruct cpy = *structPtr;
cpy.b; // 50
```

### Const Smart Pointers

Declaring the smart pointer itself `const` prevents changing the data the smart pointer points to, but does not make the data itself `const`. 
This is akin to putting `const` after the `*` for a normal pointer declaration.
To make the data constant, the template argument to the smart pointer must be `const`.

```C++
const std::unique_ptr<int> ptr = new int(5);
*ptr = 20; // good
ptr = nullptr; //error

std::unique_ptr<const int> ptr2 = new int(10);
*ptr2 = 20; //error
ptr2 = nullptr; // good
```

### Make Functions

As I've shown, the constructor for a smart pointer takes a raw pointer to take ownership of.
Now, the evaluation order of function arguments are unspecified. This is to allow,
the compiler to perform optimizations it otherwise wouldn't be allowed to do.
So consider the following:

```C++
void foo(std::unique_ptr<X> && x, std::unique_ptr<Y> && y) {}

foo(std::unique_ptr(new X()), std::unique_ptr(new Y());
```

We can think of `new` as performing two operations. Allocating memory, and calling the constructor.
So one possible execution order is the following:

1. Allocate memory for `X`
2. Construct an `X`
3. Allocate memory for `Y`
4. Construct a `Y`
5. Construct a smart pointer of `Y`
6. Construct a smart pointer of `X`
7. Call `foo`

What if the constructor to `Y` throws in step 4? Memory for `X` is already allocated!
Now we can have a memory leak!

Another downside is we are stuck manually matching the correct allocation function with deletion function.

To solve this, the C++ library has `std::make_unique` and `std::make_shared` template functions to help us. 
Whatever you pass as arguments to these functions are *forwarded* (more on this later) to the constructor of the object you are wrapping in a smart pointer. 
Like smart pointers themselves, these functions are templated on the type that they are creating a smart pointer of.

Using make functions, the allocation, construction, and adoption by a smart pointer steps occur in a single step. This prevents the
unspecified evaluation order memory leak.

```C++
auto smartPtr = std::make_unique<int>(100);
*smartPtr; // 100

auto shared = std::make_shared<MyStruct>(20, 30, -10);
shared->c; // -10
```

---

> "If debugging is the process of removing software bugs, then programming must be the process of putting them in."
>
> \- Edsger Dijkstra