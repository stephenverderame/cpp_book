# Ranges

One of tips I give at the end of this is to keep function parameters to a minimum.
Yet, the STL has plenty of functions that take quite a lot, upwards of 5 or 6.
Typically, this occurs when passing pairs of iterators because the STL was missing an abstraction: a range.

Ranges are a concept.
Each type of range corresponds to a type of iterator that satisfies the concept.
We have, as you'd expect `std::ranges::input_range`, `output_range`, `bidirectional`, `forward`, `random_access`, and `contiguous` along with `sized_range` (size can be determined in constant time),
`borrowed_range` (iterators not tied to lifetime of the object and are always safe to dereference) and a few others.
The basic requirements to construct a range is that a type provides an iterator via `begin()`, and an ending sentinel via `end()`. 

We looked at `std::string_view` before, and a range is basically just an extension and generalization of that idea. But, remember, a range is not a type, it's a concept.
Thus, a vector, map, list, anything can be used as a range. It only just needs a `begin()` and `end()` member function. 

What about pointers you ask? Well, we can wrap them in a `std::span` which takes a pointer and size, and provides the required interface for a range.
You can choose to use a span with a static or dynamic length. You can also use `std::ranges::subrange`, passing a begin and end iterator to convert traditional
C++ iterators into a range.
```C++

template<std::range R>
void func(R&& range){
    std::ranges::for_each(std::forward<R>(range), 
    [](auto& i){
        std::cout << i << " ";
    });
    std::cout << std::endl;
}

auto myStr = "Hello World";
std::span<const char> dynamicSpan 
    {myStr, strlen(myStr) + 1};

int nums[10] = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};

std::span<int, 10> staticSizedSpan {nums};

std::vector v = {20, 10, 4};
std::string str = "Hello";
const auto vRange = std::ranges::subrange(v.begin(), v.end());

func(str);
func(v);
func(staticSizedSpan);
func(dynamicSpan);
func(vRange);
```

The ranges namespace defines range version of each STL algorithm.
Actually, that's a lie. They don't operate on ranges they operate on views.
A view is something that can view or transform the underlying contents of a range.
It's important to note that operations on views are *lazy evaluated*,
the result is computed on-demand, not up-front.
Furthermore, views follow a functional paradigm: they don't mutate the actually container.

```C++
std::vector v = {10, 20, 30.0, 40.55};
for(auto& it : std::views::reverse(v)){
    std::cout << it << std::endl;
}
```
This does not create a new vector, nor does it modify the existing one.
Instead, it determines the next element, in reverse order, at each iteration.
All the views are callable objects. These objects overload `operator|` to provide pipelining capabilities. Here's an example:
```C++
auto range = v
    | std::views::filter([](auto& x){ return x % 2})
    | std::views::reverse
    | std::views::take(5);
```
This will construct a view from v, that "contains" only the first 5 odd numbers from back to front.
You can see the bitwise or operator is overloaded to work exactly like the unix pipe or pipeline operators in other languages.

Notice how each functor also has a partial application form.
`std::views::take(5)` returns another functor, this time with only 1 parameter and that's the range to operator on.
The traditional function syntax of take would be `std::views::take(v, 5)`;

Here's another example:
```C++
std::map<int, std::string> m = //...

for(auto it : std::views::keys(m)
            | std::views::sort
            | std::views::drop(10)
            | std::views::transform(
                [](auto& key){ return key * 10}))
{
    std::cout << it << std::endl;
}
```
This will take the keys of the map, sort them, skip the first 10, and for the rest: iterate over each and transform them by multiplying by 10. 

Notice that `it` is not declared `auto&`, this is because transform, and other "mutating" view transformations don't mutate anything and thus return rvalues.
You may think of a view as an immutable range of sorts.

Here's some more examples:
```C++
std::string csv = "Hello,How,House,Happiness";
std::string sentence = "Hello My name is Bob";

auto sentence1 = csv | std::views::split(',') 
    | std::views::drop(1) | std::views::take(2);
//How, House

auto sentence2 = sentence | std::views::split(' ');
//My, name, is, Bob in some order


auto isPrime = [](int x) {
    for (auto j = 2; j * j <= x; ++j) {
        if (x % j == 0) return false;
    }
    return true;
};
auto primes = std::views::iota(100)
    | std::views::filter(isPrime)
    | std::views::take(10);
// MSVC doesn't seem to like iota, but it is in the library reference
// so I'm pretty sure this is right
for (auto i : primes) {
    std::cout << i << std::endl;
}
// generate the first 10 prime numbers starting from 100.
```
`std::views::iota` can take one or two parameters.
The one parameter version creates an infinite stream starting with the value of the parameter and incremented one at a time.
The two parameter version is a finite stream of values between the first parameter and the second parameter.

Here's another example of passing a range to a function. This returns an iterator if the value was found or an empty optional if not.
```C++
template<std::ranges::range R, typename T>
auto bsearch(R&& range, T val) 
    -> std::optional<decltype(range.begin())> 
{
	auto first = std::ranges::lower_bound(range, val);
    // do not forward because we continue to use range

	if (first != range.end() && *first == val) {
		return first;
	}
	return {};
}
```