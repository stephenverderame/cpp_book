## Iterator Chain

Let's design a class to have similar syntax and semantics to Python and Rust's `chain`. What we'll do is make a more abstract version of our Rope class.
A chain will be able to chain together objects with a `begin()` and `end()` function so that they can be iterated over by one iterator.
We'll want this iterator to be `const` if any of the containers' iterators are `const`.
For simplicity, we'll make the `chain` iterator random access, however it won't truly be random access unless all the iterators it is composed of
are random access as well. This is because technically, random access iterators require `O(1)` random access; however, for iterators that don't support
that, the random access iterator methods of `chain` (such as `operator[]`) will have `O(n)` lookup time.

We actually already designed a `min_iter_tag` template which can be used to prevent the user from having to keep track of what the chain is a chain of
before using certain iterator features.

Let's see some example usage:
```C++
std::vector v = {10, 20, 30, 50};
auto init_lst = {50, 60, 70, 20};
std::list ls = {100, 200, 300};

for (auto e : chain(v, lst, ls)) {
    std::cout << e << ", ";
}
std::cout << std::endl;

auto c = chain(ls, v);
std::sort(c); // mutates ls and v

```

Let's start with a dynamic way to access tuple elements:

```C++
/**
* Returns the result of calling the specified function on the tuple element
*
* @param tup the tuple
* @param idx the index of the element to get from the tuple
* @param f the function that accepts the element at index `idx`
*   and returns a result that is returned from this function
*/
template <class Func, class Tuple, size_t N = 0>
decltype(auto) tupleGet(Tuple& tup, size_t idx, Func f) {
    if (N == idx) {
       return f(std::get<N>(tup));
    }

    if constexpr (N + 1 < std::tuple_size_v<Tuple>) {
        return tupleGet<Func, Tuple, N + 1>(tup, idx, f);
    }
    else {
        throw std::invalid_argument(
            std::format("{} is >= than tuple length {}", N + 1, std::tuple_size_v<Tuple>));
        // std::format is C++20, I'm being lazy here
    }
}
```
We've seen this before; this method just allows us to index a tuple at runtime.

Type alias helpers:
```C++
/// reference type of Containers iterator
template<class Container>
using iter_reference_t = std::remove_const_t<
    typename std::iterator_traits<decltype(std::declval<Container>().begin())>::reference>;

/// pointer type of Containers iterator
template<class Container>
using iter_pointer_t = std::remove_const_t<
    typename std::iterator_traits<decltype(std::declval<Container>().begin())>::pointer>;

/// const and reference removed T
template<typename T>
using base_type = std::remove_const_t<std::remove_reference_t<T>>;
```
SFINAE helpers:
```C++
/// value is true if Container has the same base iter type as IterType
template<typename IterType, class Container, typename = void>
struct IsSameIterType : std::false_type {};

/// value is true if Container has the same base iter type as IterType
template<typename IterType, class Container>
struct IsSameIterType<IterType, Container, std::enable_if_t<
        std::is_same_v<base_type<IterType>, base_type<iter_reference_t<Container>>>
    >> : std::true_type {};

/// true if all containers have the same iter type as FirstContainer
/// @see IsSameIterType
template<class FirstContainer, class ... Containers>
constexpr inline auto are_all_same_iter_v =
    (... && IsSameIterType<iter_reference_t<FirstContainer>, Containers>::value);

template<bool Condition, typename TrueType, typename FalseType>
struct conditional {
    using type = FalseType;
};

template<typename TrueType, typename FalseType>
struct conditional<true, TrueType, FalseType> {
    using type = TrueType;
};

/// type is HeadType
template<typename HeadType, typename ... Tail>
struct Head {
    using type = HeadType;
};

/// type is HeadType
template<typename HeadType>
struct Head<HeadType> {
    using type = HeadType;
};

/// first type in List
template<typename ... List>
using first = typename Head<List...>::type;

template<typename ... Containers>
constexpr inline auto is_any_iter_const = (... || std::is_const_v<iter_reference_t<Containers>>);
```

Class definition:

```C++
template<typename ... Containers>
class chain {
    std::tuple<Containers& ...> containers;
public:
    chain(Containers& ... containers) : containers(std::tie(containers...)) {
        static_assert(are_all_same_iter_v<Containers...>, 
            "Iterators passed to chain must have the same value type and volatility");
    }

    chain(chain<Containers...>&&) = default;
    chain<Containers...>& operator=(chain<Containers...>&&) = default;

    auto begin() const {
        return iter(this, 0, 0);
    }

    auto end() const {
        return iter(this, sizeof...(Containers), 0);
    }

    class iter {
        const std::tuple<Containers& ...>* containers;
        unsigned curContainer, curIndex;


        template<typename ... List>
        using first_iter = decltype(std::declval<first<List...>>().begin());

        using self_t = chain<Containers...>::iter;
    public:
        using value_type = typename std::iterator_traits<first_iter<Containers...>>::value_type;
        using reference = typename conditional<is_any_iter_const<Containers...>,
            std::add_const_t<iter_reference_t<first<Containers...>>>,
            iter_reference_t<first<Containers...>>>::type;
        using pointer = typename conditional<is_any_iter_const<Containers...>,
            std::add_const_t<iter_pointer_t<first<Containers...>>>,
            iter_pointer_t<first<Containers...>>>::type;
        using difference_type = typename std::iterator_traits<first_iter<Containers...>>::difference_type;
        using iterator_category = std::random_access_iterator_tag;

        template<typename DiffType>
        friend auto operator+(self_t t, DiffType diff) 
            -> std::enable_if_t<std::is_integral_v<DiffType>, self_t> 
        {
            auto res = t;
            bool keepLooping = true;
            while (keepLooping) {
                if (res.curContainer >= 
                    std::tuple_size_v<std::remove_reference_t<decltype(*res.containers)>>)
                    break;
                tupleGet(*t.containers, res.curContainer, [&](auto& tup) {
                    const auto dist = std::distance(tup.begin(), tup.end());
                    if (res.curIndex + diff >= dist) {
                        ++res.curContainer;                       
                        diff -= (dist - res.curIndex);
                        res.curIndex = 0;
                    }
                    else {
                        res.curIndex += diff;
                        keepLooping = false;
                    }
                });
            }
            return res;
        }
        template<typename DiffType>
        friend auto operator+(DiffType diff, const self_t& t) 
            -> std::enable_if_t<std::is_integral_v<DiffType>, self_t> 
        {
            return t + diff;
        }

        template<typename DiffType>
        friend auto operator-(self_t t, DiffType diff) 
            -> std::enable_if_t<std::is_integral_v<DiffType>, self_t> 
        {
            auto res = t;
            bool keepLooping = true;
            while (keepLooping) {
                if (diff > res.curIndex && res.curContainer > 0) {
                    diff -= (res.curIndex + 1);
                    res.curIndex = 
                    tupleGet(*t.containers, --res.curContainer, [&](auto& tup) {
                        const auto dist = std::distance(tup.begin(), tup.end());
                        return dist == 0 ? 0 : dist - 1;
                    });
                }
                else if (diff <= res.curIndex) {
                    res.curIndex -= diff;
                    keepLooping = false;
                }
                else {
                    throw std::invalid_argument(std::format(""));
                }
            }
            return res;
        }
        template<typename DiffType>
        friend auto operator-(DiffType diff, self_t t) 
            -> std::enable_if_t<std::is_integral_v<DiffType>, self_t> 
        {
            return t - diff;
        }




        iter(const chain<Containers...>* parent, unsigned curContainer, unsigned curIndex) :
            containers(&parent->containers), curContainer(curContainer), curIndex(curIndex) {}

        //iter() : parent(nullptr), curContainer(sizeof...(Containers)), curIndex(0) {}

        iter(const self_t& other) = default;
        self_t& operator=(const self_t& other) = default;
        iter(chain<Containers...>::iter&& other) = default;
        self_t& operator=(self_t && other) = default;

        reference operator*() {
            return const_cast<reference>(*const_cast<const self_t&>(*this));
        }

        std::add_const_t<reference> operator*() const {
            if (curContainer >= sizeof...(Containers))
                throw std::out_of_range("Chain container index out of range");
            return
                *tupleGet(*containers, curContainer, [this](auto& tup) {
                    auto it = tup.begin();
                    std::advance(it, curIndex);
                    return static_cast<pointer>(&*it);
                });

        }

        self_t& operator++() {
            tupleGet(*containers, curContainer, [this](auto& tup) {
                auto it = tup.begin();
                std::advance(it, curIndex + 1);
                if (it == tup.end()) {
                    curIndex = 0;
                    ++curContainer;
                }
                else {
                    ++curIndex;
                }
             });
            return *this;
        }

        self_t operator++(int) {
            auto cpy = *this;
            ++(*this);
            return cpy;
        }

        auto operator==(const chain<Containers...>::iter& other) const {
            return (other.containers == containers || other.containers == nullptr || containers == nullptr)
                && other.curIndex == curIndex && other.curContainer == curContainer;
        }

        auto operator!=(const chain<Containers...>::iter& other) const {
            return !(*this == other);
        }

        auto operator-(self_t other) const {
            auto cntIdx = curContainer;
            auto index = curIndex;
            difference_type diff = 0;
            while (cntIdx > other.curContainer) {
                diff += index;
                index = 
                tupleGet(*containers, --cntIdx, [&](auto& tup) {
                    return std::distance(tup.begin(), tup.end());
                });
            }
            return diff + index - other.curIndex;
        }

        bool operator<(self_t other) const {
            return curContainer < other.curContainer || curIndex < other.curIndex;
        }
        bool operator>(self_t other) const {
            return curContainer > other.curContainer || curIndex > other.curIndex;
        }
        bool operator<=(self_t other) const {
            return *this < other || *this == other;
        }
        bool operator>=(self_t other) const {
            return *this > other || *this == other;
        }

        self_t& operator+=(difference_type d) {
            *this = *this + d;
            return *this;
        }

        self_t& operator-=(difference_type d) {
            *this = *this - d;
            return *this;
        }

        self_t& operator--() {
            *this -= 1;
            return *this;
        }

        self_t operator--(int) {
            auto cpy = *this;
            --(*this);
            return cpy;
        }

        reference operator[](difference_type diff) const {
            if (diff > 0) {
                return *(*this + diff);
            }
            else {
                return *(this - std::abs(diff));
            }
        }
    };
};
```

Notice the two `operator-`. One takes another iterator, and returns the difference between the two iterators. 
The other `operator-` takes an iterator and difference type and moves the iterator similarly to `operator+`.

## Iterator Zip

Let's create a class `zip`, that can be constructed from an arbitrary amount of different containers. The definition for our
container concept is just a type that contains a semantically correct `begin()` and `end()` method.
Because I'm being a tad lazy, I'm going to forego the SFINAE enforcement of this concept.

`zip` will simply have `begin()` and `end()` methods that return a forward iterator that iterates over all the containers at once.
"De-referencing" this iterator will return a tuple of the values of de-referencing iterators to all the containers.
The iterator will just be a forward iterator, and iteration will continue until the smallest container has been completely iterated through.

Let's start with the `zip` class which will be pretty simple. We'll store a tuple of references to containers.
We must use a tuple so that we can have an unknown amount of members of different types.

```C++
template<typename ... Containers>
class zip {
    std::tuple<Containers&...> containers;
public:
    zip(Containers& ... containers) : containers(std::tie(containers...)) {}

    class iterator;

    iterator begin() {
        return iterator(&containers);
    }

    iterator end() {
        return iterator();
    }

    const iterator begin() const {
        return iterator(&containers);
    }

    const iterator end() const {
        return iterator();
    }
};
```

`std::tie` creates a tuple of lvalue references; no constructor is called for any of the tuple elements.
`std::make_tuple` will copy or move construct each element of the tuple.
We declare the `zip` iterator class, which we'll define outside the `zip` class for neatness.

Notice that since `zip` holds references to containers, it must not outlive any container passed to it.

We'll make the iterator's default constructor construct an invalid iterator.
The iterator will need access to the `zip` class containers tuple.

```C++
template<typename ... Cs>
class zip<Cs...>::iterator {
    bool invalid;
    std::tuple<decltype(std::declval<Cs>().begin())...> iterators;
    std::tuple<decltype(std::declval<Cs>().end())...> ends;
    
    using this_t = zip<Cs...>::iterator;

    template<size_t N = 0>
    bool is_end_iter() const {
        if constexpr (N < sizeof...(Cs)) {
            return std::get<N>(iterators) == std::get<N>(ends) ||
                is_end_iter<N + 1>();
        }
        else {
            return false;
        }
    }
public:
    iterator(const std::tuple<Cs&...> & context) : invalid(false) {
        std::apply([this](auto& ... containers) {
            iterators = std::make_tuple(containers.begin()...);
            ends = std::make_tuple(containers.end()...);
        }, context);
    }
    iterator() : invalid(true) {}

//...
```

We'll use an `invalid` flag to denote an iterator that reached the end of the smallest container.
Default constructing the zip iterator will set this flag to true.

We'll also store an iterator for each container, and the end iterator for each container.
Therefore, we can determine when the zip iterator is done iterating by testing if any of
the iterators equals its corresponding end iterator. This is what the `is_end_iter` method does.
If `N`, the integral template argument is less than the amount of containers we're iterating over,
then we check if the Nth iterator equals the Nth sentinel iterator. If it does, we return `true`.
Otherwise, we recurse and test the `N + 1` iterator.
If `N` is greater or equal to the amount of containers, then we simply return `false`, which is
the null element for boolean or. That is to say that `x == x || false`.

Let's look at the constructor. We use `std::apply` to unpack the elements of a tuple into
arguments of a variadic lambda. We can then use `std::make_tuple` and fold expressions to construct a tuple from
begin and end iterators. `containers` are passed by reference to avoid copies. Notice how we capture the `this`
pointer. If you remember, this is because we cannot capture member variables directly, but instead we must capture
the `this` pointer to the object they belong to.
`std::apply` is used because we cannot use a fold expression over template arguments. So the following won't compile:
```C++
template<typename ... Indicies>
auto getTuple(Indicies ... indicies) {
    // where indicies is a pack of integers
    return std::make_tuple(std::get<indicies>(iterators)...);

    // expanding a pack inside the <> is fine,
    // but expanding a pack over the <> is not
}
```

Next, our iterator needs to support equality testing. Two `zip` iterators will be equal if they are both invalid
or their `iterators` tuple is the same.

```C++
    bool operator==(const this_t& other) const {
        return invalid && other.invalid ||
            iterators == other.iterators;
    }

    bool operator!=(const this_t& other) const {
        return !(*this == other);
    }
```

For de-referencing the iterator, we'll use a similar tactic as the constructor: use `std::apply` to unpack
the `iterators` tuple as arguments to a lambda. This lambda will then dereference each iterator, and use `std::tie`
to return a tuple of references.

```C++
    auto operator*() {
        if (invalid)
            throw std::out_of_range("Attempt to derefence end iterator");
        return
            std::apply([this](auto& ... iters) {
                return std::tie(*iters...);
            }, iterators);
    }

    auto operator*() const {
        if (invalid)
            throw std::out_of_range("Attempt to derefence end iterator");
        return
            std::apply([this](auto& ... iters) {
            return std::tie(std::add_const_t<decltype(*iters)>(*iters)...);
                }, iterators);
    }
```

Lastly, we'll need to define `operator++`. Once again, we'll use `std::apply` to unpack the tuple elements as
arguments to a variadic lambda. Then we'll fold over `operator,` to increment each iterator. The comma operator in C++
is a binary operator that evaluates its left argument, then evaluates and returns the right argument.

```C++
auto i = 0;

auto j = i += 10, i++;
// j is 10
// i is 11

auto k = "Hello World", 0;
// k is 0
```

Since we only care about the side effects of the `++` operation, it doesn't matter that we discard the result of
incrementing them. After incrementing the iterators, we'll check if any of them equal their ending iterator.
If so, we'll set the `invalid` flag to `true`.

```C++
    //this_t is a type alias for zip<Cs...>::iterator

    this_t& operator++() {
        std::apply([](auto& ... iters) {
            (++iters, ...);
        }, iterators);
        if (is_end_iter())
            invalid = true;
        return *this;
    }

    this_t operator++(int) {
        auto cpy = *this;
        ++(*this);
        return cpy;
    }
```

Finally, don't forget the type aliases we need for iterators! Here's the complete code:

```C++
#pragma once
#include <tuple>
#include <stdexcept>


template<typename ... Containers>
class zip {
    std::tuple<Containers&...> containers;
public:
    zip(Containers& ... containers) : containers(std::tie(containers...)) {}

    class iterator;

    iterator begin() {
        return iterator(containers);
    }

    iterator end() {
        return iterator();
    }

    const iterator begin() const {
        return iterator(containers);
    }

    const iterator end() const {
        return iterator();
    }
};

template<typename ... Cs>
class zip<Cs...>::iterator {
    bool invalid;
    std::tuple<decltype(std::declval<Cs>().begin())...> iterators;
    std::tuple<decltype(std::declval<Cs>().end())...> ends;
    
    using this_t = zip<Cs...>::iterator;

    template<size_t N = 0>
    bool is_end_iter() const {
        if constexpr (N < sizeof...(Cs)) {
            return std::get<N>(iterators) == std::get<N>(ends) ||
                is_end_iter<N + 1>();
        }
        else {
            return false;
        }
    }
public:
    iterator(const std::tuple<Cs&...> & context) : invalid(false) {
        std::apply([this](auto& ... containers) {
            iterators = std::make_tuple(containers.begin()...);
            ends = std::make_tuple(containers.end()...);
        }, context);
    }
    iterator() : invalid(true) {}

    bool operator==(const this_t& other) const {
        return invalid && other.invalid ||
            iterators == other.iterators;
    }

    bool operator!=(const this_t& other) const {
        return !(*this == other);
    }

    auto operator*() {
        if (invalid)
            throw std::out_of_range("Attempt to derefence end iterator");
        return
            std::apply([this](auto& ... iters) {
                return std::tie(*iters...);
            }, iterators);
    }

    auto operator*() const {
        if (invalid)
            throw std::out_of_range("Attempt to derefence end iterator");
        return
            std::apply([this](auto& ... iters) {
            return std::tie(std::add_const_t<decltype(*iters)>(*iters)...);
                }, iterators);
    }

    this_t& operator++() {
        std::apply([](auto& ... iters) {
            (++iters, ...);
        }, iterators);
        if (is_end_iter())
            invalid = true;
        return *this;
    }

    this_t operator++(int) {
        auto cpy = *this;
        ++(*this);
        return cpy;
    }

public:
    using difference_type = ptrdiff_t;
    using reference = decltype(*std::declval<this_t>());
    using value_type = std::remove_reference_t<reference>;
    using pointer = value_type*;
    using iterator_category = std::forward_iterator_tag;

};
```

And here's some usage examples:

```C++
std::vector v5 = { -100, 40, 30, 700, 600, 300, 100, -50, 47, 573, 48 };
auto lst = { 'a', 'b', 'c', 'd' };
for (auto [letter, num] : zip(lst, v5)) {
    std::cout << "(" << letter << ", " << num << ")\n";
}
// (a, -100)
// (b, 40)
// (c, 30)
// (d, 700)
```
