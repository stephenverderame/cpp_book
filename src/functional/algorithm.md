# Algorithms

STL algorithms are in the `<algorithms>` header. These are generic algorithms which operate on pairs of iterators (a start and end iterator). 
C++20 includes versions of these algorithms that operate on ranges and views. This is discussed more in the C++20 section
of the book, but ranges and views are basically like an abstraction over a pair of iterators.
Often times they will take a callable object to perform some computation. 
Whenever you write a for loop, the first though should be "is there an algorithm which does this for me?". 
Generally, we want to use algorithms as much as possible because they help us avoid making simple indexing or iterator mistakes and are well tested. 
As we'll see later, some algorithms may also take an execution policy as their first argument. The execution policy changes their concurrency behavior. 
Thus, using STL algorithms, one parameter is all it takes to convert sequential code to concurrent code. 
I'll briefly discuss a few common ones, but more information can be found on the documentation of the algorithms library.

A predicate is a function that takes in an element and returns true if that element matches the criteria. 
A comparator is a function that takes in two arguments and returns true if the first argument should be ordered before the second.

If you are familiar with functional programming, some of the algorithms are very much like common operations in functional
languages such as `map` or `fold`. I indicate that in the following list with the phrase "analogous to ..." or "similar to ..."

* `std::for_each(It begin, It end, Func f)` - analogous to `iterate`
    * Calls `f`, passing in the dereferenced iterator for all elements between `begin` and `end`
* `std::for_each_n(It begin, Size num, Func)` - analogous to `take`
    * Same as `for_each` but takes a number indicating the amount of elements to iterate through starting from `begin`.
    ```C++
    std::array a = {600, 97, 3784, 474, 3, 906, 0};
    std::for_each_n(a.begin(), 3, [](auto e) {
        std::cout << e << ", ";
    });
    // 600, 97, 3784, 
    ```
* `std::all_of`/`std::any_of`/`std::none_of` `(It begin, It end, Func pred)`
    * Iterates through and applies a predicate which takes the dereference of the iterator and returns a bool
* `std::count(It begin, It end, T elem)`
    * Iterates through and counts how many elements equal `elem`
* `std::count_if(begin, end, pred)`
    * Same as `count` but counts how many times the predicate returns true
* `std::find(begin, end, elem)`
    * Returns the iterator to the first element that equals `elem` or `end` if not found
    * `std::find_if` is the same but takes a predicate instead of an element
* `std::copy(begin, end, dstBegin)`
    * Copies elements from range to a new area beginning at `dstBegin`. Does not check whether `dstBegin` is valid or can support the amount of elements in the range
    * `std::copy_if(begin, end, dstBegin, pred)` - similar to `filter`
        * Same as copy but only copies if the predicate returns true for that element
* `std::copy_backward(begin, end, dst)`
    * Same as copy but stores the elements in `dst` backwards
* `std::move(begin, end, dst)`, `std::move_backward(begin, end, dst)`
    * Self explanatory
* `std::fill(begin, end elem)`
    * Assigns `elem` to every iterator in the range
    * `std::fill_n(begin, count, elem)`
* `std::generate(begin, end, f)`
    * Assigns the result of calling `f` to every iterator in the range
    * `std::generate_n(begin, count, f)`
    ```C++
    std::vector<int> nums(100); // 100 integers
    std::generate(nums.begin(), nums.end(), [count = 0]() mutable {
        return count++;
    });
  
    std::vector<char> letters;
    std::generate_n(std::back_inserter(letters), 100, [](){ return 'A'; });
    letters.size(); //100
    ```
*  `std::transform(begin, end, [optional 2nd range begin], dstBegin, func)` - analogous to `map`, and `zip`
    * Iterates through a range or two, calls the function and stores the result of that function in the destination range
    * The second range must be at least as large as the first
    * Func takes in the underlying value of the iterator from the first range as the first argument, and if a second range is specified, the value of the second second as the second argument
    ```C++
    std::vector a = {100, 200, 300};
    std::array b = {100.0, 50.1, -300.67};
    std::vector<double> out;

    std::transform(a.begin(), a.end(), b.begin(), std::back_inserter(out),
        [](auto a, auto b) {
            return a + b;
        });
    // out is {200.0, 250.1, -0.67}

    // transform in-place is allowed
    std::transform(a.begin(), a.end(), a.begin(),
        [](auto a) {
            return a * 2;
        });
    // a is 200, 400, 600
    ```
* `std::swap_ranges(begin, end, otherBegin)`
    * Swaps the values of each element in two equal sized ranges
* `std::iter_swap(a, b)`
    * Swaps the underlying value of two iterators
* `std::shuffle(begin, end, [optional function which returns a randomly selected iterator difference_type or a UniformRandomBitGenerator])`
    * Randomly rearranges the elements
* `std::rotate(begin, new_begin, end)`
    * Rotates the elements in `begin` to `end` to the left so that the iterator `new_begin` becomes the first element of the resulting range
* `std::sample(begin, end, out, count, gen)`
    * Selects `count` elements at random from `begin` and `end` and writes them to `out` using `gen` (a random number generator) as a source or randomness
* `std::unique(begin, end)`
    * Removes consecutive duplicates from the range
    * Removal is done by moving all elements to be removed to the back of the iterator range, then returning a new ending iterator that does not include these removed elements
    * `std::unique_copy(begin, end, out)` - same but does not perform in-place
* `std::remove_if(begin, end, predicate)`
    * Removes all elements satisfying the predicate by moving them to the end of the iterator range
    * Returns an iterator to the first removed element which is also an iterator off the back of the range of remaining elements
    ```C++
    std::vector v = {100, 200, 300, 400, 500, 600};
    const auto new_end = 
        std::remove_if(v.begin(), v.end(), [](auto e) { return e <= 300; });
  
    for(auto num : v) {
        std::cout << num << std::endl;
    }
    // Prints 400, 500, 600, then the remaining numbers (probably: 100, 200, 300)
  
    for(auto it = v.begin(); it != new_end; ++it) {
        std::cout << *it << std::endl;
    }
    // Only prints 400, 500, 600 
    ```
* `std::partition(begin, end, pred)`
    * Partitions a range so that elements that satisfy `pred` are at the beginning of the range and those that don't are at the end
    * Returns an iterator to the point at which the range is partitioned
    * `std::partition_copy` - same idea but not in place
* `std::sort(begin, end, [optional comparator])`
* `std::stable_sort(being, end, [optional comparator])` - sort that guarantees that elements that compare equally are kept in the same order as they were
* `std::lower_bound(begin, end, elem, [optional comparator])`
    * Returns an iterator to the first element >= to `elem` or `end` of no such element is found
    * Requires `begin` and `end` be sorted or partitioned on `val < elem` (ascending order)
    * Think of finding the lower bound of a series of `elem` within the range
    * The comparator defines what < means in this context. Will get the first element in which calling the comparator returns `false`. The comparator returns true if the first argument is ordered before the second.
* `std::upper_bound(begin, end, elem, [optional comparator])`
    * Same as lower_bound but finds first element > `elem` or `end` if no such element is found
    * Think of finding the upper bound of a series of `elem` within the range
* `std::binary_search(begin, end, val, [optional comparator])`
    * Returns true if `val` is present, false otherwise
```C++
// Requires begin and end be sorted in ascending order
// gets iterator to first iter that equals elem or end if not found
template<typename Iter, typename T>
auto binarySearch(Iter begin, Iter end, T elem) {
    const auto e = std::lower_bound(begin, end, elem);
    return *e == elem ? e : end;
}
```
* `std::merge(begin, end, begin2, end2, out)`
    * Merges two sorted ranges and puts the result starting from the `out` iterator
* `std::make_heap(begin, end, [optional comparator])`
    * Makes the range a max heap
* `std::push_heap(begin, end, elem, [optional comparator])`
    * Appends an element to an existing heap
* `std::pop_heap(begin, end, [comparator])`
* `set_union`, `set_intersection`, `set_difference`
    * Treats two ranges as sets and performs the set operation
* `max_element`, `min_element`, `minmax_element`
    * gets the max, min, and min and max, respectively from a range
* `lexicographic_compare(begin, end, begin2, end2, [comparator])`
    * returns true if the first range is lexicographically less than the second

The following are part of the `<numeric>` header

* `std::iota(begin, end, start)`
    * Fills the range with sequentially increasing values starting from `start`
* `std::accumulate(begin, end, start, [optional operation])` - similar to `fold_left` or `reduce`
    * Sums all the elements in the range with an initial value of `start`
    * If a binary function is passed, calls that function with the first argument as the current accumulation value (initialized to `start`) and the second argument as the current value in the range. The returned value becomes the new accumulated value
    * `std::reduce` is similar to `accumulate` but it can take an execution policy, and the initial value is optional

    ```C++
    const std::vector v = {10.0, 5.4, 2, -1.2};
    const auto pi_v = std::accumulate(v.begin(), v.end(), 1, [](auto a, auto b) {
        return a * b;
    }); // 1 (initial value) * 10 * 5.4 * 2 * -1.2
    ```
* `std::partial_sum(begin, end, out, [optional op])`
    * Same as accumulate except saves each accumulation step as a separate element in `out`