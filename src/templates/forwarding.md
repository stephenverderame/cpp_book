# Forwarding

There's one type of reference we haven't looked at yet. And that's the universal, aka forwarding, reference. 
It's not exactly its own type of reference, but rather it can bind to both rvalues and lvalues and is formed like `T&&` where `T` is some template parameter. 
A universal reference cannot have qualifiers or other type adornments. For example `const T&&` is a const rvalue reference to `T` and **not** a `const` universal reference. 

For example, the following are rvalue references, not universal references:

* `const T&&`
* `std::vector<T>&&`
* `volatile T&&`
* `T*&&`

We can use universal references to create a function which works for both rvalue and lvalue references. This is possible with the help of `std::forward<T>`. 
`std::forward` is similar to `std::move` except `std::forward` conditionally casts the argument to `T&&` while move always casts its argument to `T&&`. 
`std::forward` will cast its argument to `T&&` if it's a rvalue reference otherwise it keeps it as a lvalue reference. 
This allows function that use a forwarding reference to take advantage of move semantics when they can.

```C++

template<typename T>
struct Node {
    T data;
    std::unique_ptr<Node<T>> next;

    explicit Node(T&& data) : data(std::forward<T>(data)), next(nullptr) {}
}

auto n = Node<std::string>("Hello World");
// const char * is passed by lvalue reference to Node(T&&)
// const char * used to construct std::string

auto n2 = Node<std::string>(n.data);
// lvalue reference passed to Node(T&&)
// copy constructor used to copy data

auto n3 = Node<std::string>(std::move(n2.data));
// rvalue references passed to Node(T&&)
// n3.data is move constructed from n2.data

auto getString() {
    return std::string("Hiya");
}


auto n4 = Node<std::string>(getString());
// move construct data
```

Notice that `std::forward` requires the base type to be specified as a template argument while `std::move` does not.

Overloading a function that takes a universal reference is a pretty tricky area and should be avoided. 
This is because universal references will be perfect matches for almost anything. 
You can circumvent some of these issues by using `std::enable_if` to limit the bindings on the function or use *tag dispatch* which is a technique we'll discuss later.

There are a few cases where forwarding can fail:
* Passing a brace initializer to a template. This fails because the compiler can't know if this is constructing some type or if it's a `std::initializer_list`
* Passing the C `NULL` macro. This is a macro for `0` and thus the type deduced will be `int`, not a pointer. This is why we use `nullptr`.
* Passing a function pointer that has overloads. The compiler can't tell which overload you mean to use.
* Passing bitfields, which cannot be taken by non-const reference.


Forwarding is how functions like `std::make_unique` are created. 
There's still 1 other language feature used to allow passing an arbitrary amount of arguments, but a single argument version might look something like this:

```C++
template<typename T, typename U>
auto make_unique_simple(U&& u) {


    return std::unique_ptr<T>(new T(std::forward<U>(u)));
}
```

As you can see, universal references are a template parameter. 
Which means functions using them pretty much have to be defined in a header file. 
Plus, what we'll end up with in the compiled code is a separate function for every type of parameter passed to it. This can lead to bloat in the size of the executable. 
Is there another option? Well, don't forget about passing by value. Lvalues will by copy constructed and rvalues will be move constructed. 
Then you can unconditionally move the argument since it was copied/moved. This isn't always the best choice, but it is *a* choice. 
Finally, you can also implement the rvalue reference function in terms of the lvalue reference function.

Finally, I'd like to just clarify that `std::move` should be used for rvalue references and `std::forward` for universal references.