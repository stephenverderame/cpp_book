# Variadic Templates

As we've seen with `std::void_t`, `emplace()`, and `make_unique`/`make_shared`, there is a way to take an unknown amount of parameters of different types. 
This can be done using a *parameter pack*. A template that uses a parameter pack is known as a variadic template.

You can almost think of a parameter pack as a list of template parameters (both type and non-type parameters). 
A parameter pack can have any number of parameters, including none.

```C++
template<typename ...Ts>
struct Types {};

Types<> t;
Types<1, 2, 3> t2;
Types<int, short, std::string, char> t3;

template<typename ...Ts>
void func(Ts... args) {
    // ..
}

func();
func(100, "Hello", 'g');
```

We can define a parameter pack by putting the ellipsis in front of the pack name, and expand a parameter pack by putting the ellipsis after the *pattern*. 
The pattern is the pack name with any adornments (such as `&`) which are given to all elements of the pack. During expansion, the pattern is substituted by the elements of the pack separated by commas.

```C++
template<class ...Us> 
void f(Us... pargs) {}

template<class ...Ts> 
void g(Ts... args) {
    return f(&args...); 
    // calls f by passing all elements of args by reference
    // &args is the pattern
}
g(1, 0.2, "a"); 
```

We can expand parameter packs in the following contexts:
* As we've just seen, function arguments
    ```C++
    template<typename ... Ts>
    void g(Ts... args) {
        f(args...);
        f(++args...); // expands to f(++a, ++b, ++c, ...)
        f(h(args...) + args...); 
        // expands to
        //f(h(a, b, c) + a, h(a, b, c) + b, h(a, b, c) + c);
    }
    ```
* Initializers (both parenthesis and braces)
    ```C++
    template<typename ... Ts>
    auto toVec(Ts ... args) {
        std::vector v = {1, args..., 2};
        return v;
    }
    ```
* Template arguments
    ```C++
    template<typename ... Ts>
    struct Test {
        container<int, short, Ts...> c;
    };
    ```
* Base class specifiers and member initialization lists
    ```C++
    template<typename ... Mixins>
    struct Aggregator : public Mixins ... {
        Aggregator(const Mixins& ... args) : Mixins(args) ... {};
        // expands to call copy constructor of all base classes
    }
    ```
* Using declarations
    ```C++
    template<typename ... Mixins>
    struct Aggregator : public Mixins ... {
        Aggregator(const Mixins& ... args) : Mixins(args) ... {};
        // expands to call copy constructor of all base classes

        void foo() {};
        using Mixins::foo...;
        // unshadow all base class foo overloads
    }
    ```


Parameter packs can also be universal references and can be forwarded.

```C++
template<typename T, typename ... Args>
auto make_unique(Args&& ... args) {
    return std::unique_ptr<T>( new T(std::forward<Args>(args)...) );
    // expands to
    // new T(std::forward<A>(a), std::forward<B>(b), std::forward<C>(c), /* ... */)
}
```

Basically, each use case of `...` expands a comma separated list of the pattern `...` is applied to. 
The name of the pack typename (`Args` in this case) is substituted which each type in the pack, and the name of the pack arguments (`args` here) is substituted with each argument passed to the function.

We can "unpack" an expansion by using two function overloads (or class specializations), one which takes no arguments, this will be the base case, and one which takes a "head" argument and "tail" parameter pack. 
The idea here is the same as in functional programming. Here's a simple function to get the length of a parameter pack.
```C++
constexpr auto argCount() {
    return 0;
}

template<typename T, typename ... Args>
constexpr auto argCount(T&&, Args&& ... args) {
    return 1 + argCount(std::forward<Args>(args)...);
}

constexpr auto count = argCount(5); 
static_assert(count == 1);
static_assert(argCount("Hello", 50.0, 10, 'c') == 4);
```
When we call `argCount` with non-zero amount of arguments, the first argument gets bound to `T&&` and the rest (which there may be none) gets bound to `Args&&`. 
If the pack has no arguments in it, the expansion won't do anything, and the no parameter overload will be called.

A better way to do this is to use the `sizeof...` operator, which gets the amount of arguments in a parameter pack.

```C++
template<typneame ... Args>
constexpr auto argCount(Args&& ...) {
    return sizeof...(Args);
    // sizeof... takes the typename of the pack, not the argument name
}
```

Here's an example of getting the nth type from a parameter pack:

```C++
template<unsigned index, typename Head, typename ... List>
struct NthType {
    using Type = typename NthType<index - 1, List...>::Type;
    // we "pop" the Head of the pack by not passing it through
};

template<typename Head, typename ... List>
struct NthType<0, Head, List...> {
    using Type = Head;
};
// index 0 specialization

using sndType = typename NthType<2, void*, char*, int, long, double&>::Type;
//sndType is int
/*  sndType
        NthType<index = 2, Head = void*, List = char*, int, long, double&>
            NthType<index = 1, Head = char*, List = int, long, double&>
                NthType<index = 0, Head = int, List = long, double&>
                    This is the specialization
                    So `Type = Head = int`

*/
```

## Fold Expressions

A fold expression is like another type of pack expansion, except that instead of producing comma separated arguments,
the pack is expanded over a binary operator. 
A fold expression can have an initial value as well, and must be surrounded by parentheses.

I won't explain the details of folds here, but basically left folds operate on the leftmost argument first,
right folds go from the rightmost argument to leftmost. Folds come from functional programming languages. 
Fold expression have the following syntax where `pack` is the pattern containing the name of the pack,
`op` is the operator, `init` is the initial value and `...` are the actual ellipsis.

* Unary fold right - `(pack op ...)`
* Unary fold left - `(... op pack)`
* Binary fold right - `(pack op ... op init)`
* Binary fold left - `(init op ... op pack)`

The difference between a unary and binary fold is not that one uses unary operators (that would be a normal pack expansion),
but rather the binary fold has an initial value. 
The syntax for a left fold is when the `...` is on the left side of the pack name.

```C++
template<typename ... Ts>
constexpr auto sum(Ts&& ... args) {
    return (... + args);
    // "unary" left fold
}

template<typename ... Args>
void print(Args&& ... args) {
    (std::cout << ... << args);
    // binary left fold
}

template<typename ... Args>
constexpr auto allTrue(Args&& ... args) {
    return (... && args);
    // binary left fold
}

template<typename T, typename ...Args>
constexpr auto contains(T needle, Args ... args) {
    // true if needle is contained within the pack
    return (... || (args == needle));
}

template<typename ...Args>
constexpr auto selfDot(Args... args) {
    // computes the dot product of the arguments with them self
    return (... + (args * args));
}
```

Notice the need for the parenthesis to make an entire expression such as `args * args` or `args == needle` part of the pattern. 
We define all of these function `constexpr` so that the result of the function can be available at compile time (and therefore not done during runtime) if we pass arguments that are also `constexpr` such as literals.

## Packs and Concepts

Parameter packs can be used pretty easily with `enable_if` using fold expressions.

```C++
template<typename ... Ts>
constexpr auto sum(Ts... args) 
    -> std::enable_if_t<(... && std::is_arithmetic_v<Ts>), decltype((... + args))> 
{
    return (... + args);
}

constexpr auto s = sum(10, 20.3, 100.f, 'c'); // double 229.3
```

Unlike before, in this situation we *need* to use the trailing return type because the parameter `args` is used in determining the return type. 
Also, notice how when we want a fold expression using the types of the pack, we use the name of the template parameter `Ts`. 
However, when we want a fold expression using the values of the pack, we use the name of the function argument `args`.

In this case we fold over `&&` (boolean AND) to ensure that all types in the pack are arithmetic.
We could fold over `||` (boolean OR) to check that at least one type upholds a certain condition.

We can use a type alias to make this a bit cleaner.

Another less graceful, (and pre C++17 friendly) way of doing this is to create an `all_true` struct using SFINAE. 
What we'll do is instantiate a struct with a pack of bools.
Then we'll assert that the type of the bool pack is the same when we append a `true` to the front of the pack as when we push a `true` to the back. 
If all elements of the bool pack are the `true`, then the types will be the same.
However, if any of the elements in the pack are `false`, the position of this `false` will differ between the two 
instantiations of the template, and they won't be the same type.

```C++

template<bool...>
struct bool_pack {};

// template variables are a C++17 feature
template<bool... bools>
constexpr inline auto all_true_v = std::is_same_v<
    bool_pack<bools..., true>, 
    bool_pack<true, bools...>>;

template<typename... Ts>
std::enable_if_t<all_true_v<std::is_arithmetic_v<Ts>...>>
foo(Ts... args) {
    //...
}
```

### Possible Exercises

1. Can you create a function that takes an arbitrary number of arguments and serializes all of them into a single byte array?
The function should return an `std::vector<std::byte>` or an `std::array<std::byte, N>`.
If you go the latter route, the function can be `constexpr`.
You'll need an overload to handle containers like `std::vector`, `std::list`, etc.
You may choose the endianness of the result.
    * So when passed `"Hello", 5, static_cast<short>(1000)` the function should return a single byte array that would look something like:
        ```C++
        0x48 0x65 0x6C 0x6C 0x6F 0x00 0x00 0x00 0x05 0x03 0xE8
         'H'  'e'  'l'  'l'  'o'|         5         |   1000
        // little endian, 4 byte int, 2 byte short
        ```