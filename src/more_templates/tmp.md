# Template Meta Programming

We've actually been using this technique a bit already with SFINAE. Template meta programming allows us to use template specialization to compute results as compile time. 
Fun fact: TMP was actually *discovered*, not created on purpose. For many use cases, `constexpr` functions are easier to read and write, however TMP
must still be used for "computing" types.

Let's create a TMP struct to compute the factorial of a number.

```C++
template<int num>
struct Factorial {
    constexpr int value = num * Factorial<num - 1>::value;
};

template<>
struct Factorial<0> {
    constexpr int value = 1;
};

template<int i>
using factorial_v = Factorial<i>::value;

constexpr auto f = factorial_v<5>;
```
When we instantiate the struct `Factorial<5>`, it sets the `value` member to be `5 * Factorial<4>::value`.
Then `Factorial<4>` is instantiated, and its member `value` is initialized as `4 * Factorial<3>::value`.
Then `Factorial<3>` is instantiated and its `value` member becomes `3 * Factorial<2>::value`.
This continues until we get to `Factorial<0>`, whose `value` member is initialized with the constant `1`
because of the specialization we defined.

So the general gist of TMP is to create a recursive `struct`, where the base case is implemented as a template specialization.

Computing values at compile time should be done with `constexpr` functions, but there are cases where TMP is needed. These types of problems
often require us to compute a type.
Let's say we want to return a type if a condition is true, and a different one if it's false:

```C++
template<bool condition, typename TrueType, typename FalseType>
struct Conditional {
    using type = TrueType;
};

template<typename TrueType, typename FalseType>
struct Conditional<false, TrueType, FalseType> {
    using type = FalseType;
};


template<bool condition, typename TrueType, typename FalseType>
using if_t = typename Conditional<condition, TrueType, FalseType>::type;
// we need the typename keyword because type is a member type alias
// of a template type
```
If the condition argument to `Conditional` is true, the primary definition is used and `type` is an alias for `TrueType`.
If the condition is false, the `Conditional` specialization is used and `type` is an alias for `FalseType`.

Using this, let's create a way to get the minimum iterator category from a sequence of different iterator types. We'll start with creating a
`constexpr` function that converts iterator tags to numbers so that we can order the iterator tags from least to most restricting.

```C++
/**
* Gets a number representing the functionality of an iterator where 0 is an input/output
* and higher numbers allow for more functionality.
*/
template<typename Iter>
constexpr auto get_iter_num() 
    -> std::enable_if_t<!std::is_same_v<void, typename std::iterator_traits<Iter>::iterator_category>, int> 
    // enable the function only if the type has an iterator_category in iterator_traits
    // return `int` if Iter is an iterator
{
    using T = typename std::iterator_traits<Iter>::iterator_category;
    // get tag type

    if constexpr (std::is_same_v<T, std::input_iterator_tag> || std::is_same_v<T, std::output_iterator_tag> )
        return 0;
    else if constexpr (std::is_same_v<T, std::forward_iterator_tag>) 
        return 1;
    else if constexpr (std::is_same_v<T, std::bidirectional_iterator_tag>)
        return 2;
    else if constexpr (std::is_same_v<T, std::random_access_iterator_tag>)
        return 3;
}


/// Gets the minimum iterator from a pack of iterators
template<typename Min, typename Head, typename ... Iters>
struct MinIter {
private:
    using cur_min_t = if_t<get_iter_num<Min>() < get_iter_num<Head>(),
        MinIter, Head>;
    // the "smallest" between the current Min and head of pack
    // the type with the smaller iterator number is assigned to the alias cur_min_t

    using tail_min_t = typename MinIter<cur_min_t, Iters...>::type;
    // the min of the tail of the pack
    // recursive step
public:
    using type = if_t<get_iter_num<cur_min_t>() < get_iter_num<tail_min_it>(),
        cur_min_t, tail_min_t>;
};

template<typename Min, typename Head>
struct MinIter<Min, Head> {
    // this is the base case when the tail of the pack is empty

    using type = if_t<get_iter_num<Min>() < get_iter_num<Head>(),
        MinIter, Head>;
};


/// Gets the first type in a parameter pack
template<typename H, typename ... Tail>
struct PackHead {
    using type = H;
};


/// Minimum iterator type (returns the iterator itself)
template<typename ... Iters>
using min_iter_t = typename MinIter<typename PackHead<Iters...>::type, Iters...>::type;

/// Gets the minimum iterator category
template<typename ... Iters>
using min_iter_tag = typename std::iterator_traits<min_iter_t<Iters...>>::iterator_category;
```

Let's say we wanted to chain together multiple iterators into a single iterator. It would make sense that this aggregate iterator would have the category of the weakest iterator it
is composed of. Hence, we can use TMP to query what this minimum iterator category is.

Notice the use of `using` declarations in `MinIter` to "store" the intermediate result. The using declaration is basically like the TMP version of defining variables.

We use the `typename Head, typename ... Iters` arguments to unpack the head of the parameter pack into `Head`, and the rest is put into the `Iters` parameter pack.
This allows us to pass a parameter pack and be able to query the type of the first element in the pack.

It's common to denote the TMP calculation's result type as a member named `type`, and a result value as a member named `value`. 

Let's look at another TMP technique, *tag dispatch*. The idea is to use a struct that has no members to use as a type to switch between different overloads or specializations of a function.
This is the idea behind iterator tags, such as `std::random_access_iterator_tag`. Instead of having a conditional at runtime determine which function to call, the decision can be made by the
compiler, reducing runtime cost. Let's call a different function depending on the size of a type:

```C++
template<size_t i>
struct Int2Type {
    constexpr auto value = i;
    // or
    enum : size_t { value = i };

    // just provides public access to the value
    // members are unnecessary in this example
};
// because different instantiations of a template are different types
// then different sizes passed to Int2Type<> instantiate different Int2Type types

void doFoo(Int2Type<4>);
void doFoo(Int2Type<2>);
void doFoo(Int2Type<1>);

doFoo(Int2Type<sizeof(int)>{});
// correct overload is called and inlined into the compiled code by the compiler.
```

We've seen how we can do something similar with `constexpr if` and template specialization, *tag dispatch* is just another tool in the toolbox that you may find optimal for some problem.

