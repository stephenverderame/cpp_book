# Concepts and Type Traits

In C++20, concepts and constraints are a direct language feature. However the idea of a concept has long existed before then. Consider the following function:

```C++
template<typename T, typename U>
U sum(const T& container, U initialValue) {
    // another example of making the compiler do the copy for us
    for(auto e : container) {
        initialValue += e;
    }
    return initialValue;
}
```

What is required from `T` and `U`? Well `U` needs to define `operator+=`, and that operator must take a `U` as the first parameter and whatever type is returned from dereferencing an iterator to `T`. `T` must be iterable by defining a `begin()` and `end()` function and that iterator must return a type that is implicitly convertible to `U` or `U` must provide an overload of `operator+=` so that it takes whatever is "contained" in `T`. These requirements form are the *concepts* that `T` and `U` must implement.

Templates allow for generic programming and *parametric polymorphism*, which allows a single code implementation to be used with multiple classes.

Earlier I discussed how the types passed to our `LList<T>` class needed to be default constructable and copyable. These are two major criteria for the *Regular* concept which among being default constructable and copyable also needs to be comparable with `==` and `!=`.

The `type_traits` header provides many helpers to query types or the adherence of a type to a certain quality. Templates that end in `_v` "return" a value and those that end in `_t` "return" a type. These "helpers" are not actually functions but rather templated `constexpr` variables or type aliases in `struct`s. Therefore, the results of these queries are available at compile time.

```C++
std::is_default_constructable_v<std::string>; // true
// is_default_constructable is a template struct
// so the above is shorthand for:
std::is_default_constructable<std::string>::value;

std::is_fundamental_v<char>; // true
std::is_arithmetic_v<int>; // true
std::is_same_v<int, const int>; // false
std::is_floating_point_v<long long>; // false
std::is_signed_v<unsigned>; // false

std::rank_v<int(&)[3]>; // 1 - dimensions of array type

std::is_base_of_v<Base, Derived>; // true

using typ = std::remove_cv_t<const bool>; // bool
// once again, shorthand for
std::remove_cv<const bool>::type;

enum class myEnum {}
std::underlying_type_t<myEnum>; // int
```

[cppreference](https://en.cppreference.com/w/cpp/header/type_traits) has a list of all of them.
Now how do these work? I won't explain the full story just yet, but they can use template specialization behind the scenes. Let's create a simple `is_bool` type trait.

```C++
template<typename T>
struct is_bool {
    inline constexpr static auto value = false;
    // inline so redefinitions are allowed in a program
};

template<>
struct is_bool<bool> {
    inline constexpr static auto value = true;
};

template<typename T>
inline constexpr auto is_bool_v = is_bool<T>::value;
```

`is_bool` is specialized so that when `T` is `bool` its `value` member will be true. Now what about `const bool` or `bool&` or `const bool&`? Since these are distinct types from `bool`, they'll fall into the unspecialized version of `is_bool` and `value` will be false. One solution would be to pass the argument into `std::remove_reference_t` and `std::remove_cv_t` prior to passing it to `is_bool_v`.

```C++
template<typename T>
using strip_t = std::remove_cv_t<std::remove_reference_t<T>>;
// order matters here, remove_cv removes the top level const and volatile qualifiers
remove_cv_t<int * const volatile>; // int* because const and volatile apply to the pointer
remove_cv_t<const volatile int *>; // const volatile int * because const and volatile apply to the data

is_bool_v<strip_t<const bool &>>; // true
is_bool_v<const bool>; // false

template<typename T>
inline constexpr auto is_bool2_v = std::is_same_v<T, bool>;
```

There's a whole bunch more of them that can be found [here](https://en.cppreference.com/w/cpp/header/type_traits). Now that we have these traits, the most basic approach to enforcing adherence of template parameters to a concept is with `static_cast`.

```C++
template<typename T, typename U, typename V>
V sum(T a, U b) {
    static_cast(std::is_arithmetic_v<T> && std::is_arithmetic_v<U>,
        "Sum passed non arithmetic types");
    return a + b;
}
```

This incurs no runtime penalty since the check happens during compilation.

A better approach is to use `std::enable_if_t`. As the name might suggest, the main usage of this is to conditionally enable functions. How this works is by "returning" the second template type parameter if the first template parameter evaluates to a true condition at compile time. Then you can specify the return type of the function to be an `enable_if_t` expression. If the function is called with template parameters that make the condition true, all is well. Otherwise, the enable if effectively prevents the instantiation of the function template causing the compiler to complain that the function doesn't exist. Not as clear as an error message as you'd get in C++ 20 with concepts and constraints, but trust me it's far nicer than some of the monstrosities you might receive without it.

The default type for the second template argument is `void`. 

```C++
template<typename T, typename U>
constexpr inline auto are_arithmetic_v = std::is_arithmetic_v<strip_t<T>> && 
    std::is_arithmetic_v<strip_t<U>>;

template<typename T, typename U, typename V>
std::enable_if_t<are_arithmetic_v<T, U>, V>
sum(T a, U b) {
    return a + b;
}
// same thing with trailing return type
template<typename T, typename U, typename V>
auto sum2(T a, U b) -> std::enable_if_t<are_arithmetic_v<T, U>, V>
{
    return a + b;
}
/*
This is not type deduction, the auto doesn't mean auto like normal
but instead its syntac for trailing return type
the return type of the function is specified after it using a ->
allows the parameters to be used in the return type
*/
```

A trailing return type is most useful with `decltype()` which "returns" the type of the expression passed to it. The expression passed is not evaluated, but instead the compiler figures out what the type of the expression is and replaces the `decltype` expression with this type during compilation. Thus it incurs no runtime penalty.


```C++
template<typename T>
auto getIter(const T& container) -> decltype(container.begin()) {
    return container.begin();
}
// Of course the trailing return type here isn't necessary

template<typename T>
auto getIter(const T& container) {
    decltype(*container.begin()) acc = 0;
    // auto will deduce to int, but we want the type to be whatever container holds
    for (e : container) {
        acc += e;
    }
    return acc;
}
```

So how does all this work? By a language feature known as *SFINAE* which stands for specialization failure is not an error. Basically, if instantiating a template causes an error, the compiler will look for another template specialization. If no other template is found, it complains that it can't find the class or function.

I think it's best to start with an example.
```C++
template<typename T, typename = void>
struct SFINAE : std::false_type {};
// primary definition
// 2 template arguments, the second is unnamed and defaulted to void

template<typename T>
struct SFINAE<T, std::void_t<typename T::type>> : std::true_type {}; 
// 1 template argument specialization
// instantiates if T has a member type alias called type

enum e {};

static_assert(SFINAE<std::underlying_type<e>>::value); 
//good (second declaration) bc std::underlying_type<e> has a type alias called type

static_assert(SFINAE<std::string>::value); 
// error (first declaration) bc std::string does not have an alias type
```

Let's walk this through. First we create our default struct. We have it inherit from `false_type` which gives it a constant member `value` that is false. This is the primary definition. The `typename = void` is somewhat odd, but all it's doing is creating a template with two parameters, where the second one is unnamed and defaults to `void`. This allows code to reach this version of the struct `SFINAE` if the specialization fails. Then we create a specialization. The specialization subtypes `std::true_type` to get a `constexpr` member `value` that's true.

For the second template argument of the specialization, we pass `T::type` to `std::void_t`. If the type arguments of `std::void_t` (multiple can be passed) are valid, `std::void_t` is substituted with `void`. If it isn't, then specialization fails. We need the `typename` keyword whenever we are accessing a type alias that's part of a template type.

```C++
struct MyStruct {
    using name = int;
};

template<typename T>
auto test() {
    typename T::type t_type; // accessing alias of template parameter, need typename keyword
    typename std::vector<T>::iterator it; // std::vector<T> is a template, need typename keyword
    std::vector<int>::iterator it2; // no keyword needed
    MyStruct::name i; // also no keyword
}
```

So if `T` contains a type alias called `type`, then `std::void_t` will be substituted with `void` and the specialization will succeed. Otherwise, the specialization fails and the next best specialization is chosen. When we use these structs, we'll only pass 1 type parameter. Thus, the compiler will first try to instantiate the better matching specialization (1 template parameter) before trying to instantiate the next best match (2 template parameters with one of them defaulted).



```C++
template<typename T, typename = void>
struct IsThreadDestructionPolicy : std::false_type {};

template<typename T>
struct IsThreadDestructionPolicy<T,
	std::void_t<
        decltype(T::onThreadDestroy(std::declval<std::thread&>()))
    >> : std::true_type {};
```


Think of `std::void_t<... Ts>` as "try to instantiate" the following types. In actuality, `void_t` is a usage of SFINAE itself. In this case, what we try to instantiate is the return type of the function `onThreadDestroy`. It's static and takes a reference to a thread. In order to get the proper function signature, we must somehow "construct" an object of the type it expects and pass this object to it at compile time. That's basically what `std::declval<T>()` does. `std::declval` can not be ODR-used, so it is only applicable in unevaluated contexts such as template arguments. Note that to get a reference of `T`, like we need here, you must explicitly request it by putting an `&` in the brackets. `Sizeof()` is another unevaluated context: it does not actually evaluate whatever expression you pass to is.

Now what if for some type `T`, it doesn't have the function `onThreadDestroy`? Well, we get a specialization failure, and by the name of SFINAE this does not halt compilation, rather it just goes on to the next possible instantiation which is the struct that inherits from `false_type`. This is why our primary definition needed that second parameter. We need a parameter to put the `std::void_t` to see if we can resolve a correct type. If we can't, because this second parameter is defaulted, the compiler will then choose the less restricting primary definition of the struct. It won't choose this first, because our users will only supply one type parameter and a specialization taking one parameter will be chosen over one in which it takes a second, but the second is defaulted.

Let's create a struct to check if a type implements an iterable concept. We'll check that a type has `begin()` and `end()` member functions and that whatever returned from those functions is incrementable, dereferenceable, and comparable with `==` and `!=`. As we'll soon see, there's a better way to check for this and these requirements don't even cover all our bases.

```C++
template<typename T, typename = void>
struct IsIterable : std::false_type {};

template<typename T>
using iter_t = decltype(std::declval<T&>().begin());

template<typename T>
struct IsIterable<T, std::void_t<
        decltype(std::declval<T>().begin()),
        decltype(std::declval<T>().end()),
        decltype(*std::declval<iter_t<T>>()),
        decltype(++std::declval<iter_t<T>>()),
        decltype(std::declval<iter_t<T>>()++),
        decltype(std::declval<iter_t<T>>() == std::declval<iter_t<T>>()),
        decltype(std::declval<iter_t<T>>() != std::declval<iter_t<T>>()),
    >> : std::true_type {};

template<typename T>
constexpr inline auto is_iterable_v = IsIterable<T>::value 
    && std::is_default_constructible_v<iter_t<T>>
    && std::is_copy_constructible_v<iter_t<T>> 
    && std::is_copy_assignable_v<iter_t<T>>
    && std::is_swappable_v<iter_t<T>>;

template<typename T>
auto printContainer(const T& container) -> std::enable_if_t<is_iterable_v<T>>
{
    std::cout << "[";
    for (auto e : container) {
        std::cout << e << ", ";
    }
    std::cout << "]\n";
}

const std::vector v = {10, 20, 30, 40};
const std::string s = "Hello World";
const char * str = "Hiya";

printContainer(v);
printContainer(s);
printContainer(str); // error!
```

Why is this useful? Well, try uncommenting the trailing return type with `enable_if_t` and calling `printContainer(100)`. See what error message you get. Trust me it gets way worse when there's template functions being called in template functions and where they're being used in multiple source files across a project that has thousands of lines of code. In visual studio, the linter will be also able to pick up on violating `enable_if` the second you type the function call.

 
Here's another example
```C++
template<typename T, typename = void>
struct IsThreadInterruptionPolicy 
: std::false_type {};

template<typename T>
struct IsThreadInterruptionPolicy<T,
    std::enable_if_t<
        std::is_same_v<T, InterruptableThreadPolicy> || 
        std::is_same_v<T, UninterruptableThreadPolicy>
    >>
    : std::true_type {};
```
Here we use `std::enable_if` which is a struct that defines a member `type` if the first template parameter is true. If the first argument to `enable_if` is `false`, then it doesn't have a member alias named `type` and the specialization will fail. `std::enable_if`and `std::void_t` work very similarly, the difference is `enable_if` takes a boolean or boolean expression while `std::void_t` takes a type or list of types. Instead of having to type `typename std::enable_if</*...*/>::type` or `typename std::void</*...*/>::type` we use `std::enable_if_t` and `std::void_t` respectively. Of course in C++17 this whole example can simply be written as:

```C++
template<typename T>
constexpr inline auto is_thread_interruptable_policy_v = 
    std::is_same_v<T, InterruptableThreadPolicy> || 
    std::is_same_v<T, UninterruptableThreadPolicy>;
```


Let's see another example of SFINAE. Here it's used to ensure that only certain types are passed and to get a friendlier compiler error if these conditions are violated. 
```C++
/**
* Enables a function at compile time if all 
* the type parameters are integral types or vectors
* @param <T> typename to check if it is an integer or 
* vector type
* @param <ReturnType> defaults to std::vector<uint8_t>, 
*   the return type of the function if enabled
*/
template<typename T, 
typename ReturnType = std::vector<uint8_t>>
using requires_int_or_vec = std::enable_if_t<
    is_int_or_vector<strip_t<T>>::value, ReturnType>;



template<typename T>
static constexpr auto convertToByteArray(const std::initializer_list<T>& numArray) 
    -> requires_int_or_vec<T> 
{
    std::vector<uint8_t> ret;
    ret.reserve(numArray.size() * sizeof(T));
    for (auto& v : numArray) {
        const std::vector<uint8_t> subArray 
            = convertToByteArray(v);
        ret.insert(ret.end(),
            subArray.begin(), subArray.end());
    }
    return ret;
}
```
This will only compile if `T` is aN int or vector/array like type. `is_int_or_vector` is a custom struct defined using SFINAE similar to my earlier example.

If C++ 20 is available to you, use C++ 20 concepts and `requires` clauses.

Let's recap:
* `decltype` - gets the type of whatever expression is passed to it. Like `sizeof`, the expression passed is not actually evaluated but simply used for the compiler to figure out the type of said expression.
* `std::declval<T>` - creates a "proto" `T` object that can be used in unevaluated contexts like `decltype`. Allows us to "call" member functions or functions taking a `T` in such unevaluated contexts. `T` need not be default constructable. If you needed `T` to be default constructable as well, you could use `T()`, however this would always return a prvalue.
* `std::void_t<T, ...>` - if all the types passed as template arguments are valid, substitutes the entire expression with `void`. Otherwise it fails to instantiate causing whatever template specialization it's used in to also fail to instantiate.
* `std::enable_if_t<Condition, Type>` - if `Condition` is true, substitutes the entire expression for `Type`. Otherwise, it fails to instantiate causing whatever function it's used in to also fail to instantiate.

Let's look at one final example from our `sum` function we motivated this section with. This will build on the `IsIterable` example:

```C++
/*
Original Function:

    template<typename T, typename U>
    U sum(const T& container, U initialValue) {
        for(auto e : container) {
            initialValue += e;
        }
        return initialValue;
    }
*/

template<typename T, typename U, typename = void>
struct IsSummable : std::false_type {};

template<typename T, typename U>
struct IsSummable<T, U, std::void_t<
    decltype(std::declval<T&>() += std::declval<U>()),
    decltype(std::declval<U&>() += std::declval<T>()),
    decltype(std::declval<T>() + std::declval<U>()),
    decltype(std::declval<U>() + std::declval<T>()),
>> : std::true_type {};

template<typename Container, typename Acc>
constexpr inline auto is_vector_summable_v = is_iterable_v<Container> &&
    IsSummable<decltype(*std::declval<iter_t<Container>>()), Acc>::value;


template<typename T, typename U>
auto sum(const T& container, U initialValue) 
    -> std::enable_if_t<is_vector_summable_v<T, U>, U> 
{
    for(auto e : container) {
        initialValue += e;
    }
    return initialValue;
}
```

### Possible Exercises

1. Use SFINAE to ensure that the previous template classes are protected from misuse.
