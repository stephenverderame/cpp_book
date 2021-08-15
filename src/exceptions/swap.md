### Exercise From Last Time

```C++
string EvaluateSalaryAndReturnName( Employee e )
{
    if( e.Title() == "CEO" || e.Salary() > 100000 )
    {
      cout << e.First() << " " << e.Last()
           << " is overpaid" << endl;
    }
    return e.First() + " " + e.Last();
}
```

1. title of `e` is `"CEO"` and we enter the if
2. title of `e` is not `"CEO"` but salary of e is `> 100000` and we enter the if
3. neither of the above is true and we don't enter the if

Now, here's the exceptional paths of execution:

4. `e` is passed by value. Thus its copy constructor may throw
5. `e.Title()` may throw
6.  `e.Salary()` may throw
7. `==` may be an overloaded function and may throw
8.  `"CEO"` might be implicitly converted to whatever is returned by `e.Title()` (likely a `std::string`) and the conversion could fail
9. `>` might be a user-supplied function and throw
10. `100000` might be implicitly converted to whatever is returned by `e.Salary()`
11. `||` might also be a user supplied function and throw
22. `e.First()` may throw, or it may return something that must be implicitly converted to something printable
13. `e.Last()` has same reasoning as above
14. Any of the five `<<` may throw

And lastly:

19. In the final return `e.First()` may throw
20. In the final return `e.Last()` may throw
21. `" "` likely has to converted to string or whatever is returned by `e.First()` and `e.Last()`. This conversion can throw
22. the first `+` has to construct a new string which may throw
23. the second `+` may also throw for the same reasoning

This is from GotW 20 and Exceptional C++. In writing this out right now, I still missed one (the `||`). 
23! That's a lot for 5 lines of code. If we wanted to be pedantic, we could even count more.

# Swap

Consider that you are writing a RAII class. How can we make the following exception safe?

```C++
MyString& operator=(const MyString& other) {
    delete[] buffer;
    buffer = new char[other.size()]; //We'll talk about this later
    _size = other._size;
    std::copy(other.buffer, other.buffer + other._size, buffer);
    return *this;
}

~MyString() {
    delete[] buffer;
}
```
There's quite a few problems with this. First, what if we self-assign (assign the same object to itself)? 
We'd destroy the memory, then try to copy from the memory we just deleted. 
Of course we can add a check for self-assignment, but that's what considered a "code smell", or code that "smells" funny and likely indicates a flaw in the design. 
We can also see that if the allocation fails, the buffer is still deleted; the state of the program is still changed!
The failed allocation will not only put the object in a broken state, but the object's the destructor will try and delete the memory again! 
A solution could be to assign the buffer to `nullptr` right after the call to `delete[]` and check for `nullptr` in the destructor. 
Furthermore, what if the copy fails? We won't break any invariants, but it would only give us a basic guaranteed copy operator.
```C++
// basic guarantee with anti-patterns
MyString& operator=(const MyString& other) {
    if(this == &other) return *this;
    delete[] buffer;
    buffer = nullptr;
    buffer = new char[other.size()];
    _size = other.size;
    std::copy(other.buffer, other.buffer + other._size, buffer);
    return *this;
}

~MyString() {
    if (buffer) { // if buffer is not nullptr
        delete[] buffer;
    }
}
```
Applying the above fixes would give us a constructor that satisfies the basic guarantee but can we do better? Yes we can! With the *copy-and-swap idiom*!

```C++
// Not entire class definition
class MyString {
    char * buffer;
    friend void swap(MyString&, MyString&) noexcept;
// note you could define the function right here 
// instead of declaring it and defining it separately
public:
    MyString() : buffer(nullptr) {}

    MyString& operator=(MyString other) {
        // notice other is now passed by-value!

        // make a copy, noexcept swap with the copy
        swap(*this, other);
        return *this;

        // strong
    }
    // Move constructor just for fun
    MyString(MyString&& other) noexcept
        : MyString() // call default constructor to initialize
    {
        swap(*this, other);
    }

    MyString(MyString other) : MyString() {
        swap(*this, other); //same idea as operator=
    }

    ~MyString() {
        if (buffer) {
            delete[] buffer;
        }
    }
}
void swap(MyString& a, MyString& b) noexcept {
    using std::swap;
    
    swap(_size, other._size);
    swap(buffer, other.buffer);
    // just swaps the pointers, cheap
}
```
Let's first turn our attention to `operator=`. 
We are passing by value and letting the compiler do the copy for us via the copy constructor. If you need to copy something, a good optimization is to pass it by value and let the compiler handle it. 
This might not work for every situation, but it works here! Do we have to pass by value? No. We could pass by reference and do the copy ourselves as well.

Now, if the copy fails, we're all good; no state was changed. 
Next we just swap the internals, which we declared noexcept. Therefore, we got our code up to the strong guarantee! 
Like move operations, `swap` should be `noexcept`. Beyond the exception reasoning, `noexcpet` swap and move operations enable them to be used in STL containers like `std::vector`.

Swap is often implemented either as a nonmember friend function, a member function, or via a template specialization to `std::swap`. 
The problem with the member function route is that it's not idiomatic: the STL containers won't take advantage of the custom swap function; furthermore swap seems more natural taking two arguments. 
A template specialization of `std::swap` is viable, but has a few flaws which we'll see later when discussing templates. 
I tend to go the nonmember friend route as that allows users that use `swap` on our class to take advantage of our `swap` function instead of the standard implementation. 
The standard implementation isn't too bad: it's implemented in terms of move assignment but will require the introduction of one extra variable to store the intermediate step between the moves. 

Now notice the use of `using std::swap` in our swap function. The point of this is to take advantage of ADL. 
As a reminder, ADL states that if a specific namespace qualification isn't specified for a function, it will first look in the namesapce of the arguments of that function. 
Thus the purpose of `using std::swap` is to provide a fallback on the standard's default implementation if a type does not define a swap function. 
Therefore, unlike most cases where it is good practice to qualify standard library functions with `std::`, it's better to leave swap unqualified and manually bring in `std::swap` into the smallest scope possible as shown above.

### Possible Exercises

1. A RingBuffer is a fixed size buffer that loops around on itself. If you append an element past the end of the buffer, that element overwrites the first element in the buffer. Can you make a RingBuffer that stores `int` using `new[]` and `delete[]`? 
The class should:
    * Have a constructor taking the size of the RingBuffer to make
    * Be copyable (strong) and moveable (noexcept)
    * Overload `operator[]` to be read only indexable (strong)
        * If an out of bounds index is passed, take the modulo of that index and return the element at that location
            * Throw an error if that location has yet to be set
        * Note: taking the modulo of a negative number in C++, and many programming languages, results in a negative number
    * Have a `push_back` function that will overwrite previous elements as earlier described (strong)
    * Have a `pop_back` function which returns the last element and removes it (strong)
        * Copying primitive types cannot fail
    * Define `size()` which gets how many valid elements are in the buffer (noexcept)
    * Have a `noexcept` nonmember swap
    * You can augment this interface in any other way you'd like

    ```C++
    int * buffer = new int[size];
    delete[] buffer;
    ```