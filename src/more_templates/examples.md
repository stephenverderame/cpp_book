# Iterator Chain

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

Notice the two `operator-`. One takes another iterator, and returns the difference betweeen the two iterators. 
The other `operator-` takes an iterator and difference type and moves the iterator similarly to `operator+`.