# Move Optimizations

Still with me? Some of the last chapter definitely got pretty deep in the weeds. In the spirit of full disclosure, that's because during my Googling to double check what I did know, I found quite a bit I didn't know. I realized that I had no idea when it was useful to return by an rvalue reference aside from re-implementing `std::move` and also that my understanding of xvalues was pretty muddled. It just goes to prove my point that I'm no expert. Anyway...


Earlier I said that a simple function such as:

```C++
Person getPerson() {
    return Person("Lily");
}
```

*could* cause 2 copies in pre-C++11 and 2 moves today. Well, I doubt any commercial compiler would actually perform those 2 moves. Consider the following code:

```C++
class Test {
public:
    Test() {
        std::cout << "Init\n";
    }
    Test(const Test&) {
        std::cout << "Copy\n";
    }
    Test(Test&&) {
        std::cout << "Move\n";
    }
};


auto getTest() {
    return Test();
}

auto t = getTest();
```
On MSVC and GCC the result is simply `"Init"`. Not a single copy or move is performed. This is known as the RVO or Return Value Optimization and it allows (pre C++17) or guarantees (C++17 and later) the elision of the two aforementioned temporaries when a *prvalue* is the expression in the return statement. This can happen **even if elision changes the behavior of the program**. As you can see, the RVO changed the behavior of the program from printing
> Init
> Move
> Move

to just "Init". So even if the constructor had some code performing a side effect, that move can be elided. Now for RVO to be mandatory, the prvalue being returned must have the exact same type as the return type of the function. Also recall that prvalues are not polymorphic. So doing something like this:

```C++
Test getTest() {
    Test t;
    return std::move(t);
    // xvalue, not prvalue
    // type is Test&& not Test
}
```

makes it ineligible for RVO. The RVO also works when a function has multiple return statements.

Now what about something like this:

```C++
Test getTest() {
    Test t;
    return t;
}
```

According to our rules this doesn't qualify for RVO since `t` is an lvalue. But, if you run this code, chances are you'll notice once again only "Init" is printed. This is known as the Named Return Value Optimization or NRVO. NRVO is **not mandatory** however its a common optimization performed by compilers. Basically, its RVO but for values that have a name. Other non-mandatory copy elision can occur in a throw statement and in a catch clause when the thrown exception has the exact same type as the exception in the catch clause. When this elision occurs, the exception is essentially caught by reference, however this elision won't introduce polymorphism.

Let's play compiler; how could we elide these temporaries? Well, we needs to construct the return object directly in the caller's stack. So, maybe the caller could pass us a pointer to an area of memory that the return value can fit in, and we can just construct the object in that area of memory. 

```C++
Test getTest() {
    return Test();
}

auto t = getTest();

// could be turned into something sort of like this

void getTest(void * returnValue) {
    new (returnValue) Test();
    // construct new Test at the pointer given
    // actual RVO by compilers probably wouldn't
    // do it this way
}

// sizeof() returns a compile time constant
char buffer[sizeof(Test)];
// can't have an array of Test object or
// pass a Test pointer because we aren't the ones constructing a Test object
getTest(buffer);
Test * t = reinterpret_cast<Test*>(buffer);

// yes this is a pointer and not a value, but this is just the general idea
// and not even close to an actual implementation
t->~RVO();
```

