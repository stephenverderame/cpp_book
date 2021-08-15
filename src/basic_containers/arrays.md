# Arrays

Let's start with the modern array. A C++ array is a fixed-sized container of elements of a fixed type stored contiguously is memory. 
Unlike Java, the size of the array must be known at compile time and cannot change. This because the memory for an array is allocated on the stack statically.
While this means that arrays aren't as flexibly as a dynamically sized container, it also means that the memory allocations are much more efficient.
`std::vector` would be your go-to dynamically sized container in C++. Arrays are a class, and part of the STL (standard template library). 
For now, think of a template like a Java Generic or simply as a type parameter that can be supplied to classes or functions. 
This allows us to create one class for the concept of an array, and *instantiate* that class on any type we want such as `int` or `std::string` or a custom user-defined type. 
Templates are a big topic we'll discuss later. The interface for the array is declared in the `array` header and therefore must be included via `#include <array>`.

```C++
#include <array>


int main() {
    std::array nums = {1, 2, 3, 4, 5};
    nums.size(); //5
    nums[0]; //1
    nums[4] = 20;

    for(auto n : nums) {
        std::cout <<  n << " ";
    }
    // prints "1 2 3 4 5 "
    return 0;
}
```

We could not declare `num` to be `auto` because by default a list of elements surrounded by braces will be deduced as something else called an `std::initializer_list`. 
After `nums` has been created, we cannot change the size of `nums`, or the type of elements `nums` store.

In fact, the full definition of `nums` in reality looks like this: `std::array<int, 5> nums = //...`. 
But the compiler is smart enough to deduce the type and size of `nums` since we initialize it with values. 
If we did not have the data we want to store in the array at the ready, we would have to specify the size and type to be stored in it.

```C++
std::array<int, 3> nums;
nums.empty(); // false - array has elements, we just didn't set them
std::array<int, 0> nums2;
num2.empty(); // true

std::array<char, 5> letters('a', 'b', 'c', 'd', 'e');
std::array<double, 3> dbls = {3.14, 2.17, 6.28};
const std::array<std::string, 5> names(/*...*/);

for(auto n : letters) {
    // ...
}
```

An array is *iterable* (more on that later) and therefore we can use an *enhanced for-loop* to iterate over it. The for loop shown above will go through each element in the array, copy the value into `n`, and then execute the code in the for loop. This is much less error prone then using the C style indexing loops. If we didn't want to copy the value, we could have declared `n` to be a reference like so:
`for(const auto& n : nums)`. And since we aren't modifying the data, it's good practice to declare the reference `const` as well.

We can also have arrays of arrays as well.

```C++
std::array<std::array<int, 3>, 3> twoDArray = {
    {1, 2, 3},
    {2, 2, 3},
    {3, 2, 3}
};

twoDArray[0][1]; //2
twoDArray.front().front(); //1
twoDArray.back(); // {3, 2, 3}
twoDArray.empty(); // false
```

Why are arrays so limiting? Well the compiler does not dynamically allocate arrays like Java. The memory is allocated on the stack like other variables with automatic lifetimes, which is much more efficient then dynamic allocations. 
Obviously, there are times when dynamic allocation is necessary and for those times we have the `std::vector`.

## C Array

Before looking at more STL containers, first I want to take some time to look at the C array. 
The C array is a more primitive version of the C++ STL array. Like its STL counterpart, the size must be known at compile time. 
However, the STL array is a class, which has member functions such as `size()` while the C array does not. Instead, we can use the `sizeof()` operator to get the size of the array **in bytes**.

```C++

int nums[] = {1, 2, 3};
float vertices[27];

sizeof(vertices) / sizeof(float); //27
vertices[0] = 0.1f;

int classes[3] = {2112, 3110, 4280};
```

`sizeof()` can get us the size of a variable or type, and the compiler does this by looking at the declaration of the variable and seeing how much space that variable takes up. 
It should be noted that `sizeof()` includes padding, so the `sizeof()` a struct or class may be greater than the `sizeof()` all the members because the compiler may add padding to align the data to make it more efficient to use.

C arrays have another problem: they are implicitly convertible to a pointer to the first element of the array and will *decay* into a pointer when it is passed by value. 
When an array decays into a pointer, the compiler loses the dimensionality information of the array and instead of returning the size of the array, `sizeof()` will return the size of the pointer.

```C++

int nums[3] = {0, 0, 0};

sizeof(nums); // (4 or 8) * 3


void useArray(int nums[] /* same as: int * nums */) {
    std::cout << sizeof(nums); // sizeof(int *) (typically 4 or 8)
}

useArray(nums);
```

Passing an array by value (`int nums[]`) is actually the same as passing a pointer by value `int * nums`. We say that an array *decays* into a pointer.
However, like function pointers, since you didn't manually allocate this pointer you should not manually free it. 
Once decayed into a pointer, it is no longer possible to use `sizeof` to compute the size of the array. This is because the compiler is longer keeping track of that information.
We can prevent this decay by using modern arrays, or by passing an array by pointer or reference.


```C++

// pass by const reference to array of size 3
void useArrayRef(const int (& myArray)[3]) {
    std::cout << sizeof(myArray) / sizeof(int);
}

// pass by const ptr to array of size 3
void useArrayPtr(const int (* myArray)[3]) {
    std::cout << sizeof(*myArray) / sizeof(int);
}

int nums[3];
nums[0] = 1;
nums[2] =  10;

sizeof(nums) / sizeof(int); // 3

useArrayRef(nums); // 3
useArrayPtr(&nums); // 3
```