# The Rest

I think we covered enough ground to make the rest of the containers pretty self explanatory. I'll point out a few things in each and if you want more details I suggest checking out their respective chapters in A Tour of C++ or their documentation on cppreference or cplusplus.com.

One thing I will point out is that `std::string`'s `find()` member is the oddball, returning an index or `std::string::npos`. The other containers' `find()` method returns an iterator to the element or the sentinel which is returned by `end()`. Some containers don't have a `find()` member function. For those you can use `std::find()` passing in an iterator the beginning and end of the range in the container you want to search. We'll talk more about STL algorithms like `std::find()` later.

### `std::vector`

Stores elements contiguously in memory. The class typically just contains a pointer to the first element and size of the vector.

Accessing an invalid index using `operator[]` is undefined behavior but doing the same using the member function `at()` with throw an `std::out_of_range` exception.

As we discussed in the last chapter, appending elements to the back of a vector is cheap since capacity grows at a rate of `2x` giving us an amortized cost of `O(1)`. Likewise popping elements off the back or removing a range of contiguous elements that includes the last element is also cheap because the vector can just decrease the value of its `size` member. It likely won't even free the memory yet because common use cases tell us that it is very likely the user will add more element to the vector.

Removing or inserting elements in the middle or beginning of the vector, on the other hand, is quite expensive. The vector will have the move all the elements coming after the insertion/removal point forward or backwards, respectively, the same amount of spaces as elements removed or inserted.

As with `std::string`, you cannot index an element that is off the back of the vector (index `>= size()`) but still in the allocated area of memory (index `< capacity()`) because that memory is not necessarily initialized.

`std::vector<bool>` differs greatly from `std::vector` because it does not adhere to the interface exactly. For space optimization reasons, an `std::vector<bool>` is basically a dynamic bitset, which each element taking up one bit. Functions like `operator[]` and `at()` do not return references to elements anymore, because you cannot take the address of a bit. Instead they return proxy objects for manipulating the bit at the specified index. 

```C++
// Most containers:
std::vector ints = {1, 2, 3};
int* data = &ints[0];
int* end = data + ints.size();

for(; data != end; ++data) {
    std::cout << *data << std::endl;
} // 1 2 3

// std::vector<bool>
std::vector<bool> bools = {true, false, true};
bool* data = &bools[0]; // error
bools.size() * sizeof(bool); // not the amount of memory taken up!
```

### `std::bitset`

`std::bitset` is basically an array of bits. The size is a template parameter like `std::array` and must be known at compile time.

```C++
std::bitset<32> nums;
nums.reset(); // set all to 0
nums.set(); // set all to 1
nums.flip(); // flips each bit

nums[0] = 1; // no bounds checking

nums.set(2, 1); // set bit at index 2

nums.size(); // 32, num bits

nums.count(); // 2, number of set bits

nums.test(2); // 1, gets bit at index specified

if (nums.all() || nums.none()) {
    // if all bits set or no bits set
}

if (nums.any()) {
    // if at least 1 is set
}

std::string bitString = nums.to_string();
unsigned long u32 = nums.to_ulong();
unsigned long long u64 = nums.to_ullong();
```

### `std::map`

Implemented with balanced binary search trees (typically Red-Black Trees). `std::map` stores key value pairs, so an iterator to an `std::map` returns `std::pair<K, V>` where `K` is the key type and `V` is the value type. A pair has `first` and `second` member variables for accessing the respective element.

If the key passed to `operator[]` (which can be a non-integer) does not exist in the map, it is created using the default constructor along with a default constructed value. `at()` on the other hand will throw an exception if the element doesn't exist.

### `std::set`

Also implemented with balanced binary search trees. Stores unique elements.

The default implementation of `std::map` and `std::set` use `std::less<T>` to compare the elements. Therefore keys must be comparable by `operator<` or specialize `std::less<T>`. You can also specify a different comparison functor type that overloads `operator()` as the second template parameter. We'll discuss this more later.

For both `std::map` and `std::set`, finding an element uses `operator==` to compare for equality.

### `std::multiset` and `std::multimap`

Same as their respective non-multi variants except they can hold duplicate keys.

### `std::unorded_set` and `std::unordered_map`

Same as their respective ordered variants except implemented with a hash map / hash set. Due to the hash function, an iterator will traverse the map/set in an arbitrary order while their ordered variants will traverse the keys in order from least to greatest.

The keys of each must specialize `std::hash<T>` to provide a hash function if they are a user defined type or you must change the third template argument to a type of a functor to perform the hashing. Keys are compared via `operator==` which can be changed by changing the 4th template argument to a custom comparator type which overloads `operator()` to perform the comparison.

### `std::forward_list`

Typically a simple linked list implementation. Performs constant time insertion/deletion from anywhere in the list. Can only iterate through it one way (forward). Linear time random access.

### `std::list`

Typically a doubly linked list. Again performs constant time insertion / deletion at any point in the list. Iterators are bidirectional. Linear time random access.

### `std::deque`

"Double ended queue". Performs fast insertions and deletions at both ends of lists. Typically implemented as a sequence of small chunks of contiguous allocated elements. Constant time random access but linear insertion or removal of elements in middle of deque. Due to extra bookkeeping, slower indexed access than `std::vector` and a larger size of the class.

### `std::stack`, `std::queue`, and `std::priority_queue`

These are not containers themselves but *adapters*. They provide an interface of their respective data structure using the container specified as the second template argument. `std::stack` and `std::queue` use `std::deque` as their default container while `std::priority_queue` uses `std::vector`.

The comparator for `std::priority_queue` is it's third template argument. The priority queue cannot change the priority of existing elements. Internally, the priority queue uses functions like `std::make_heap`, `std::push_heap`, `std::pop_heap`, and `std::sort_heap` which manage a heap given a starting and ending random access iterator.

### `std::initializer_list`

This one isn't technically part of the containers either. The purpose is to allow you to pass an arbitrary amount of the same type arguments to a constructor or function. The initializer list can be set, and iterated and that's about it. 

```C++
auto sum(std::initalizer_list<int> list) {
    auto sum = 0;
    for (e : list) {
        sum += e;
    }
    return sum;
}

const auto nums = {1, 2, 3, 4}; // type deduction of {} is initializer list
sum(nums); // 10


class MyClass {
    std::vector<int> nums;
public:
    MyClass(std::initializer_list<int> startingNums) : nums(startingNums) {}
};
```