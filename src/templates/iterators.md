# Iterators

There are actually multiple different iterator concepts. All iterators must be copy constructable and copy assignable, destructible, provide a prefix `operator++` for incrementing and an `operator*` for dereferencing. Futhermore, all iterator lvalues must be able to be swapped and all iterators must provide type aliases through `std::iterator_traits`. This is a struct that is specialized for iterators to provide the following member type aliases:
* `difference_type` - result of subtracting iterators
* `value_type` - resultant type of dereferencing the iterator and passing by value. So qualifiers such as `const` would not be included here.
* `pointer` - pointer to `value_type` with qualifiers
* `reference` - reference to `value_type` with qualifiers
* `iterator_category` - tag indicating the type of iterator

The type aliases can be declared by specializing `std::iterator_traits<T>` directly or by declaring them as public member type aliases in the class of the iterator itself. `std::iterator_traits<T>` is another usage of SFINAE. If `T` has the aforementioned type aliases, then `std::iterator_traits<T>` will be a struct that contains the exact same type aliases. Otherwise, `std::iterator_traits<T>` will not define any type aliases. Therefore, we could make an `IsIterable` trait before like so:

```C++
template<typename T, typename = void>
struct IsIterable : std::false_type {};

template<typename T>
struct IsIterable<T, std::void_t<
    typename std::iterator_traits<decltype(std::declval<T>().begin())>::value_type
    >> : std::true_type {};
```

An implementation of `std::iterator_traits` might look something like this:

```C++
template<typename T, typename = void>
struct iterator_traits {};

template<typename T>
struct iterator_traits<T, std::void_t<
    typename T::value_type,
    typename T::difference_type,
    typename T::pointer,
    typename T::reference,
    typename T::iterator_category
>> {
    using value_type = typename T::value_type;
    using difference_type = typename T::difference_type;
    using pointer = typename T::pointer;
    using reference = typename T::reference;
    using iterator_category = typename T::iterator_category;
};
```
* Base Iterator Requirements
    * Copyable
    * Destructible
    * Able to be dereferenced and incremented
* Input Iterator
    * Can be compared for equality with `==` and `!=`.
    * Can be dereferenced to an rvalue
* Output Iterator
    * Can be dereferenced to an lvalue
* Forward Iterator
    * Input or Output Iterator that can be default constructed and is *multi-pass*
    * Multi-pass: dereferencing and incrementing the iterator doesn't affect the ability for the iterator to dereference. Basically, using the iterator does not consume anything. The iterator can increment to the sentinel and start over and do it again without a change in behavior.
* Bidirectional Iterator
    * Forward Iterator that can be decremented
* Random Access Iterator
    * Bidirectional Iterator that supports arithmetic operations such as `+` and `-` to move the iterator
    * Also supports `operator-(Iterator, Iterator)` which returns the distance between two iterators
    * Can be compared with `<`, `>`, `<=`, and `>=`
    * Supports compound assignment operators `+=` and `-=`
    * Supports `operator[]` such that `it[n] == *(it + n)`.
* Contiguous Iterator
    * Random Access Iterator with its elements adjacent to each other in memory
    * Requires that `*(it + n) == *(std::addressof(*it) + n)`

```
Iterator
|
|_ Contiguous Iterator
        |
        |_ Random Access
                |
                |_  Bidirectional
                        |
                        |_  Forward
                                |
                                |_  Input
                                |
                                |_  Output
```

Each iterator has a corresponding iterator tag type that would be assigned to the `iterator_category` type alias. The tags are `std::input_iterator_tag`, `std::output_iterator_tag`, etc.

The STL provides some general methods for operating with iterators that are specialized to use more efficient operations if available. `std::advance(it, diff)` takes an iterator reference as its first parameter and a different type as its second parameter. It will advance the iterator `diff` spaces using `operator+` if it's available otherwise will repeatedly increment the iterator. `std::distance()` will return the distance between two iterators. Similarily, `std::distance` will be constant time for iterators that support `operator-(iter, iter)`. The STL also provides `std::next` and `std::prev` which are similar to `std::advance` for incrementing and decrementing the iterator.

When designing generic code, it's best to be **as generic as possible**. For example, if you're using iterators in a for loop, it's better to use `operator!=` rather than `operator<` for comparing the iterator to the sentinel because only Random Access and Contiguous iterators support `operator<`. This doesn't just apply to iterators. Any time you use a type parameter you ideally want to use the most broadly applicable interface possible to reduce the requirements for that type.

Consider:
```C++
// Needs a random access iterator and a container with a size member and operator[]
template<typename T>
void reverse(T& container) {
    for (auto i = 0; i < container.size() / 2; ++i) {
        using std::swap;
        swap(container[i], container[container.size() - 1 - i]);
    }
}

// Needs a forward iterator
template<typename T>
void reverse(T& container) {
    const auto size = std::distance(container.begin(), container.end());
    const auto it = container.begin();
    for (auto i = decltype(size){0}; i < size / 2; ++i) {
        using std::swap;
        auto a = it;
        auto b = it;
        std::advance(a, i);
        std::advance(b, size - i);
        swap(*a, *b);
    }
}

```

Yes, the more generic version is more code and can be ever so slightly less efficient. But unless you have a good reason not to, it's generally more preferred to be as generic as possible. Another good example would be using `size() == 0` to check for emptiness of a container. Some containers don't have a concept of `size` but still have a concept of emptiness which can be tested with the `empty()` member function.

Let's look at an example of creating our own iterator
```C++
/**
* A class which strings together two vectors into one
* without copying them
*/
template<typename T>
class Rope {
private:
	std::vector<T> first, second;
public:
    // construct a rope from two vector rvalues
	Rope(std::vector<T>&& first, std::vector<T>&& second) : 
        first(std::move(first)), second(std::move(second)) {}

	class iterator {
		Rope<T>* owner;
		size_t pos;
	public:
	    // type aliases for all iterators
		using value_type = std::remove_cv_t<T>;
		using difference_type = ptrdiff_t;
		using reference = T&;
		using pointer = T*;
		using iterator_category = std::forward_iterator_tag;

		iterator(Rope<T>* owner, size_t pos) :
			owner(owner), pos(pos) {};


		bool operator==(const iterator& other) const {
			return owner == other.owner && 
			    pos == other.pos;
		}
		bool operator!=(const iterator& other) const {
			return !(*this == other);
		}
		T& operator*() {
			if (pos >= owner->first.size() && 
				pos < owner->second.size() + owner->first.size()) 
			{
				return owner->second[pos - owner->first.size()];
			}
			else if (pos < owner->first.size())
				return owner->first[pos];
			else
				throw 
				    std::out_of_range("Iterator out of range");
		}
		iterator& operator++() {
			++pos;
			return *this;
		}
		// postfix increment
		iterator operator++(int) {
			iterator old = *this;
			operator++();
			return old;
		}
	};
    // iterator to start of container
	iterator begin() {
		return { this, 0 };
	}
	// iterator to end
	iterator end() {
		return { this, first.size() + second.size() };
	}
};
```

As you see here, iterators are often implemented using pointers. So they must not live beyond the lifetime of the container they reference. `iterator` is a subclass of `Rope<T>`. So the fully qualified name for the iterator is `Rope<T>::iterator`. We cold also have forward declared the iterator inside the class and defined it outside the class. Later we'll see an implementation of `chain` which allows stringing together an arbitrary amount of different types of iterators.

In C++ 20, the typename `iterator_category` in `std::iterator_traits` is replaced with `iterator_concept` and the `std::iterator_tag` is replaced with iterator concepts. 

## Iterator Adapters

Iterator adapters wrap existing iterators in another iterator to provide certain functionality. These are mostly self-explanatory, and I won't go through every single one here.
* `reverse_iterator` - wraps an existing bidirectional iterator into an iterator such that `++` of the reverse iterator calls `--` of the underlying iterator. Returned by `rbegin()` and `rend()` of standard containers
    *  Can be created with `std::make_reverse_iterator()`
* `move_iterator` - dereferences to an rvalue to allow moving the value referenced by the iterator
    * Factory function `std::make_move_iterator()`
* `back_insert_iterator` - output iterator that appends to a container using that containers `push_back()` method
    * Factory function `std::back_inserter`
* `front_insert_iterator` - Can you guess what this does? Yup, uses `push_front()` instead of `push_back()` but otherwise quite similar to `back_insert_iterator`
    * `std::front_inserter`


### Possible Exercises

1. Make an iterator for the `Ringbuffer` class
2. I recommend checking out [this GOTW](http://www.gotw.ca/gotw/018.htm). A little old, but still plenty relevant.