# Concepts

Concepts are probably my favorite C++ 20 features (so far).
I showed you an example before but now I'm going to take some more time to explain them.
Concepts are essentially "meta-types". They provide a type system to templates.
Here's the motivation: not every template function can do something meaningful for every type.
Instead of producing strange error messages when a type that doesn't provide the expected interface is used
(In GCC I once got one over a couple hundred lines, which is rookie numbers),
the compiler will fail a little more gracefully.
We "solved" this already with SFINAE and CRTP, but concepts provides greater language support.

Let's start with the `requires` clause, aka *constraints*.
The `requires` keyword can be added after the function declaration or the template declaration,
and imposes some constraint on the types that the template parameter can bind to.
It takes a constant boolean expression, that when false, prevents that type from binding.

```C++
template<typename T>
    requires std::is_arithmetic_v<T>
T add5(T t) {
    return t + 5;
}

auto r = add5(20);
auto bad = add5("Hello"); //error!
```
As you can see, this dovetails nicely with pre C++20 type traits. 
The `requires` clause can be passed any constant, boolean expression.
When the evaluation of this expression is `false`, the function or class instantiation will fail to be generated.
The constant expression doesn't have to be one involving a template type.

This is all well and good, but for complicated type checks, this is quite annoying.
That's why we can define our own concepts.
A concept is a special type of constant boolean expression (in the TS, you'd put `bool` before `concept` in the definition).
In fact, concepts *do* evaluate to booleans, so you can use them anywhere you can use a `bool`. One way to define them is as such:

```C++
template<typename T>
concept MyConcept = std::is_arithmetic_v<T> || 
                    std::is_pointer_v<T>    ||
                    std::is_enum_v<T>;
                    
template<MyConcept MC>
void do(MC m);
// do only binds with types that statisfy MyConcept

template<typename T>
    requires MyConcept<T>
void do2(T m);
// do2 requires all types to satisfy MyConcept

template<typename T>
void do3(T m) requires MyConcept<T>;

static_assert(MyConcept<int>); 
// use it as a bool

template<typename T>
concept MyConcept2 = MyConcept<T> &&
    std::is_signed_v<T>;
// Define a concept in terms of another
```
A concept can be used as the "type's type" in a template argument, or can be used in a constraint.
In this example the concept is set equal to some constant expression, and we use it by having it as a constraint
via `requires` or as the `typename` keyword in a template argument list.

Concepts are templates themselves, and the first template type argument is the type the concept is being tested on.

Let's see a little more complex examples.
```C++
template<typename T>
concept SingletonDeadReferencePolicy = requires(T a) {
	T::onDeadReference();
};
// type must have a static function 
// onDeadReference()

template<template <typename> typename T, typename R>
concept SingletonCreatePolicy = requires(T<R> a) {
	{T<R>::create()} -> std::same_as<R*>;
	{T<R>::free(std::declval<R*>())} noexcept;
};
// type must have a static function create() the returns an R*
// must also have a static, noexcept free() function that
// takes an R*


template<typename R, typename T>
concept StaticCastableTo = requires(T a) {
	static_cast<R>(a);
};
// checks if T can be cast to R
// notice the use of a, which is an "instantiation" of the type T
```

Concepts can have constraint blocks, which are blocks that start with `requires(...)`.
These constraint blocks are passed instances of the template arguments of the concept.
Constraint blocks are similar to the practice of using `std::void_t` to ensure that a type has certain type aliases, or instance or static members.
Since concepts are checked at compile time, no real instance is actually passed to the constraint block,
instead these instances serve just like the "instances" we created with `std::declval<>`.
Therefore, we use these instances to check that a type has certain instance members or can be passed to certain functions.

The constraint block will contain a series of expressions that are unevaluated.
Type `T` satisfies the concept if all expressions are valid.
We can also check the return type of these expressions satisfy a specific concept by wrapping them in `{}` and followed then with `->`.
To the right of the arrow, we put the concept that we want the return type of the expression to satisfy. 
Standard concepts such as `std::same_as` are defined in the `<concepts>` header.


Concepts can also be used to check for member type aliases, and we can nest other concepts as well by putting a `requires` clause
inside the constraint block.

```C++
template<typename T>
concept TList = requires(T list) {
	typename T::Value;
	typename T::Next;
};
// Type must have a member type alias called Value
// and Next

template<typename T>
concept MyConcept = requires(T a) {
	{ typename T::Value } -> std::same_as<int>; 
	//requires to have a typedef/using for int

	{a.member} -> std::derived_from<Base>; 
	//requires instance member named member derived from Base

	{a.otherMember} -> std::same_as<bool&>;
	// otherMember is type bool

	{a.function()} -> std::is_convertable_to<int>; 
	//function that returns something that can be an int
	
	requires std::derived_from<TBase>;
	// type type must derive from TBase
}
```

Concepts are excellent for PBD and templates. Prefer concepts over SFINAE.
The same rules for when to use `template` and `typename` everywhere else in C++
also applies to using `template` and `typename` in constraints.
So, generally speaking, if we want to check a type has a member type alias, we use
the `typename` keyword and if we want to check a type has a template method,
we need the `template` keyword.