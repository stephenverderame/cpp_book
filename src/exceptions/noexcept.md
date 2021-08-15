## Noexcept

We saw how `std::exception::what()` is defined as `noexcept`. This means that it cannot throw an exception, or a thrown exception indicates a fatal error. 
Any exception that propagates out of a `noexcept` function calls `std::terminate` and ends the program. 
So you should declare functions that do not throw, or only throw to indicate program-terminating failures (like memory allocation failures), as `noexcept`. 
Furthermore, a `noexcept` function permits the compiler to optimize away error handling infrastructure for that function. So it's a good idea to declare everything that you can as `noexcept`.

```C++
int add(int a, int b) noexcept {
    return a + b;
}
```

We can also make functions *conditionally noexcept*. This means that the function will be `noexcept` if and only if some compile time expression is true.

```C++
void x() noexcept; //noexcept

template<typename T>
void f() noexcept(sizeof(T) < 4); //noexcept if sizeof(T) < 4

void k();

constexpr bool isNoexcept = noexcept(k()); //false
// using noexcept() has a conditional operator
// noexcept(f) is true if f is a noexcept callable object
// otherwise the result is false

bool t() noexcept(noexcept(f<char>)); //t is noexcept
// noexcept(f<char>) is a constant expression that evaluates to true since f<char> is noexcept
// because sizeof(char) < 4

bool s() noexcept(noexcept(f<double>)); //s is not noexcept

bool d() noexcept(noexcept(k())); //d is not noexcept

constexpr auto testNoexcept = noexcept(t()); //true
```
Notice how when we test if a function is noexcept, we need to have two `noexcept`s. This is because the inner `noexcept` is a conditional test, which returns `true` or `false` if the function passed is `noexcept` or not. 
The outer noexcept takes a `constexpr` boolean expression, and makes a function `noexcept` if the condition is `true`. 
Thus, when we pass the expression `sizeof(T) < 4`, we only need one since this is a `constexpr` expression. 

We'll talk about templates and `constexpr` more later, but basically all of these conditions are determined at compile time. This incurs no runtime cost.

## Exception Guarantees

There are 3:

* Nothrow Guarantee

    This is denoted by `noexcept` on the function declaration. This is the strongest exception guarantee, and it doesn't necessarily mean the function does not throw. 
    What it means is that if the function does throw, some disastrous and non-recoverable error just occurred. Use this more often than you might think, since "dead programs tell no lies"

* Strong Guarantee

    This guarantees that an operation is atomic with respect to exceptions. The operation either completes, or an exception is thrown and nothing happened. 
    If an operation with this guarantee fails, the state of the program will be exactly what it was before the operation.

* Basic Guarantee

    This guarantees that if an operation does fail, no invariants are broken and the program will return to some valid state. 
    This is the weakest of the three, and the minimum guarantee required for a correct program.

C++'s STL always offers the strong guarantee (or noexcept when denoted as such) except for range insertions (offers the basic guarantee), 
insert and erase on deque and vector (which are strong only if their elements' copy and assignment offer that guarantee), and the streams library (which only offers the basic guarantee).

It's good to document when a function has the strong or basic guarantee. No need to document nothrow in most cases since `noexcept` is self-documenting.

Destructors and move operations should be `noexcept`.

```C++

void logExn(const char * information, const std::exception & caughtExn) {
    std::cout << "An exception has occurred!"
        << " (" << information << ")\n"
        << caughtExn.what()
        << std::endl;
    //operator<<(std::ostream&, _) has the basic guarantee
    //so this function will only have the basic guarantee
    // Ex. An error could occur printing caughtExn.what() after
    // "An exception has occurred" has already been printed
    // furthermore, an error could occur on outputting "occurred!"
    // after "An exception has" may already have been displayed
}

void logExnImproved(const char * information, const std::exception & caughtExn) {
    std::stringstream ss;
    ss << "An exception has occurred!"
        << " (" << information << ")\n"
        << caughtExn.what();
    const auto msg = ss.str(); //convert to string
    // If any of the above fails, state remains unchanged

    std::cout << msg << std::endl; 

    // Slightly improved by buffering the message first
    // but still basic
}

/// Appends c to both outStr and outVec
void appendToBoth(std::string & outStr, std::vector<char> & outVec, char c) {
    outStr += c; // strong
    outVec.push_back(c); // strong
    /* This code is broken
     An error may occur appending c to outVec after c was already appended to outStr
     This violates the postcondition and hence our function is broken
    */
}

/// Gets a tuple of (strBuffer, vecBuffer) after c has been appended to both
auto appendToBoth2(const std::string & strBuffer, const std::vector<char> & vecBuffer, char c) {
    std::string strCpy = strBuffer; // copy ctor, strong
    std::vector vecCpy = vecBuffer; // strong
    strCpy += c;
    vecCpy.push_back(c);
    return std::make_tuple(strCpy, vecCpy);

    // Now we achieved the strong guarantee!
    // It cost 2 extra copies and a few extra moves but this isn't something to worry about
    // worry about performance only if a profiler tells you to
}

// Member function of some queue class

    /// Peek and pop the front of the queue
    /// Requires the queue is not empty
    T ppop() { 
        queue.pop(); // strong
        return t;
        // Pre C++11, this return would invoke a copy, which may throw
        // if this threw, then we would completely lose the front element since it had already been popped from the queue
        // Post C++11 this invokes a move which is noexcept

        // So for our purposes, this function is strong
    }
```

Exception safety shouldn't be an afterthought. We'll return to this topic again later.

Before we move on, how many paths of execution can you find in this code segment? (Answer in next chapter, from Herb Sutter's GOTW and Exceptional C++).
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

