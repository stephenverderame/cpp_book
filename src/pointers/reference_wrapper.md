# Reference Wrappers

Earlier I alluded to some limitations of pure C++ references: they must always be initialized, most containers cannot hold 
references, and references cannot be rebound.

Here's an example:
```C++
class MyList::iterator {
private:
    MyList& parent;
    // we must either set the value of a reference when its declared
    // or set the reference in a constructor initializer list (done below)
    //  if the reference is a class member

    size_t index;
public:
    iterator(MyList& owner, size_t idx) : parent(owner), index(idx) {}

    iterator& operator=(const iterator& other) {
        // ...
    }
};

// ...

    MyList a = //...
    MyList b = //...

    MyList::iterator it = a.begin();
    it = b.begin();

```

References must either be assigned a value when they are declared or, in the case of references as class members, they
can also be initialized in a constructor initializer list.

Now an iterator is something you'd expect to be able to be copied. But reference members (along with `const` members) prevent the use of compiler generated copy operations. To see why, try to come up with a way to define `operator=` that makes sense.

You might first think to define it like so:

```C++
iterator& operator=(const iterator& other) {
    index = other.index;
    parent = other.parent;
    return *this;
}
```

But now, what happens when we assign `it = b.begin()`? Well, we actually change the data of `a` to be that of `b`
through the line `parent = other.parent`.
So instead of copying iterators, we ended up copying the entire list and overwriting `a`'s data.

So references cannot be copied. This is one of the reasons (along with the inability to have pointers to references) that 
containers can't store references.

This doesn't mean that if we need references with value semantics that we are forced to use old-style pointers. We can use `reference_wrapper`.

A `reference_wrapper` acts like a non-owning pointer. It can be rebound and copied. It's interface is quite simple, 
you can use the `get()` member method to get a reference to the data it wraps, 
and if it holds a reference to a callable object, you can invoke `operator()` on the `reference_wrapper` directly.
A `reference_wrapper`, like a smart pointer is actually a template class where its template type is the type of the
reference it holds.

```C++
int n = 5;
std::reference_wrapper n_ref = n;
// same as: 
// std::reference_wrapper<int> n_ref(n);

int n2 = 10;
auto n2_ref = std::ref(n2); 
// convenience function for creating reference wrappers

n_ref = n2_ref;
n_ref.get() = 20;
// n is still 5
// n2 is now 20
n2_ref.get(); // 20


auto foo(int x) {
    return 5 * x;
}

auto foo_ref = std::ref(foo);
foo_ref(10); // 50
```

As seen in these examples, instead of calling the constructor directly, its more common to use the `std::ref` function
to create a `reference_wrapper`. This function is a template too, but the compiler can easily deduce the 
template type parameter based on the type of the argument we pass to it.

As a quick aside, I want to mention the type of `foo_ref`; it's `std::reference_wrapper<int(int)>`.
The `int(int)` is the type of a function that returns an `int` and has one integer argument. You may recall
that the type of a function pointer to such a function would be the same thing with an added "`(*)`": `int(*)(int)`.
This is very much like every other type, however for functions we need the parentheses around the asterisk so we know
that it's a function pointer and not a function returning an `int *`. So then `void()` would be the type of a function
that has no arguments and no return value, and `std::string(const char *, const std::vector<char>&)` would be the type
of a function that returns a string and takes a `const char *` and reference to a vector of characters.

Returning back to `reference_wrapper`: let's say, in the spirit of const correctness, that we don't want to change
the data the reference refers to and so we want to make the reference wrapper `const`. The first attempt at this might
be to just add `const` to the type declaration like a normal reference.

```C++
const auto n = 10;
const auto m = 20;
const auto r = std::ref(n);

r = std::ref(m); //compiler error!

r.get() = 50; // and this still works!
```

But when we do that, we're making the `reference_wrapper` itself be `const`, and not the data it refers to.
This may be desireable in some situations when we don't want to be able to rebind the reference through
`operator=`, but it isn't at all what we intended here.

Just like smart pointers, the solution here is to make the template type be `const`. So instead of
`reference_wrapper<int>`, we want `reference_wrapper<const int>`. Intuitively you can think of this
as the data being constant versus the reference itself (which has its own data stored in a different
are of memory) being constant.

The standard library provides a nice `std::cref()` function for constructing const reference wrappers.
As with smart pointers, `std::reference_wrapper<int>` and `std::reference_wrapper<const int>` are *not*
the same type.

```C++
int n = 2;
auto r = std::cref(n);
int m = 5;

r.get() = 10; //error, const 

r = std::cref(m); // works

auto j = std::ref(n);

r = j;
// This does a bit more than before
// This first uses an implicit conversion constructor and make a
//  new std::reference_wrapper<const int>
// Then it will call operator= on this newly created object
//
// As we'll see later, move semantics and rvalue reference overloads
// allows this process to be relatively efficient.
```

The standard library does a pretty good job making templates of `const` and non-const types, along with 
templates of polymorphic types behave like normal via conversion constructors, but it's important to remember
that these templates do not have the exact same relationship they would if we used references directly.

```C++
auto foo(std::reference_wrapper<int>);
auto bar(std::reference_wrapper<int>&);

int n = 10;
auto rf = std::ref(n);
auto crf = std::cref(n);

foo(rf);
foo(crf);
// works, in both cases we construct a new 
// std::reference_wrapper<int>
// which is relatively cheap

bar(rf);
bar(crf); 
//^ doesn't work since crf is a reference_wrapper<const int>
```

`std::reference_wrapper` does no lifetime checking, and its up to you to ensure that the data it refers to
is not destroyed while the reference is in use. The main advantage `reference_wrapper`s have over raw pointers
is that they are unambiguous. It's very clear that a reference wrapper doesn't own the data it refers to,
and a reference wrapper to an array will have a different type than a reference wrapper to a single
value. But to reiterate, reference wrappers have no mechanisms to know if they're still valid.