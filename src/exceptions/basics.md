# Exceptions

C++ provides the ability to `throw` and `catch` exceptions. Actually, you can throw and catch any value.
When you throw something. The stack *unwinds* until it reaches a caller which handles the exception.

```C++
try {
    throw "Hello";
} catch (const char * msg) {

}

```

The C++ standard library provides the base class `std::exception` which other standard exceptions are derived from and which you can use as a base class for user defined exceptions. 
`std::exception` provides a virtual function `what()` which is used to return a C string that contains a description of the error that occurred.

```C++
class TextureLoadException : public std::exception {
    std::string errorMsg;
public:
    explicit TextureLoadException(const std::string & file) {
        errorMsg = "Could not create a texture from the image at: " + file;
    }

    const char * what() const noexcept override {
        return errorMsg.c_str();
    }
};

// from RAII example: in constructor of LoadImage

    img = stbi_load(file, &width, &height, &channels, NULL);
    if (!img) {
        throw TextureLoadException(file);
    }

```

In this example, we may throw an exception in a constructor. Is that safe? Yes! And in fact it's recommended.
If the object cannot be constructed, throwing an exception is often the best solution.

When catching exceptions, typically we often want to catch exceptions by reference. This allows us to catch exceptions polymorphically. 
So if we catch `const std::exception &` we can catch `std::exception` and all its subtypes without object slicing. 
Furthermore, when catching polymorphic exceptions, we want to catch the most derived type first and least derived type last. 
If we did it the other way, derived types will be caught by the super type catch block preventing the code from ever reaching the derived type catch block.

```C++
try {
    // do something
} catch (const std::invalid_argument &) {
    // invalid_argument derives from logic_error

} catch (const std::logic_error &) {

} catch (const std::exception & e) {
    e.what(); // polymorphic what()
} catch (...) {
    // catch anything
}
```

Exceptions do incur a cost. So they should be used for *exceptional* circumstances, not things that are expected to happen or commonplace. 
For example, using an exception to signal the end of a file is probably not the best choice because it would be raised every time a file is read. 
For a situation like this, an `std::optional` would be a better choice which is something we'll discuss later.

C code typically uses error codes to indicate errors. This is **not** a good practice in C++, although it is still prevalent. Why?

* It's far cleaner, and cleaner code is easier to reason about

    Compare 
    ```C++
    const uint8_t size = narrow_cast<uint8_t>(
        sensor.parser.expectedLength());
    const auto resp = sendFrameAndCheckResp(
        fmt->formatQueryMsg(sensor.message_id, size, motor), size);
    auto ret = 
    sensor.parser.parse<double>({ resp.begin(), resp.end() });
    return ret;
    ```
    with
    ```C++
    const uint8_t size = narrow_cast<uint8_t>(
        sensor.parser.expectedLength());
    if (auto msg = fmt->formatQueryMsg(sensor.message_id, size, motor), size)) {
        if (const auto resp = sendFrameAndCheckResp(msg)) {
            auto ret = sensor.parser.parse<double>(
                { resp->begin(), resp->end() });
            return ret;
            // callee must verify return isn't nullptr
        } else
            //some error handling..
    }
    else
        // some error handling...
    ```
    We just doubled the amount of code, and we didn't even get to whatever would need to be done for error handling. Next, the callee must then check the return isn't
    nullptr, so they don't dereference null.
    
* <u>It's just as efficient or more efficient.</u> Yes there's some overhead to exceptions. But it's minuscule, and besides "Premature optimizations are the root of all evil". 
And furthermore, exceptions are designed to be exceptional. That means that the compiler can easily optimize for the common case where no exceptions are thrown. 
Even with branch prediction, excessive if statements are going to cause a slowdown as well.
    
* <u>Callee's can't ignore them.</u> In the previous example, we required the callee to check the return value to make sure it wasn't nullptr. Are they going to do that? Maybe, maybe not. 
With exceptions, they either handle it, or they don't. But unlike returning nullptr or error codes, if they don't handle it, the exception safely propagates out of the callee. Remember, "dead programs tell no lies." 

## Assertions

Assertions are a way to cause an error if some condition isn't true. In C++, we have the `static_assert()` which tests if a `constexpr` expression is true. 
If it isn't, compilation halts with a compiler error, and a user defined error message if one is specified.

```C++
static_assert(sizeof(int) == 4, "Requires that integers are 32 bits");

static_assert(sizeof(short) != sizeof(int)); //no user defined message
```

The C standard library also provides an `assert()` macro in `asssert.h`. This performs runtime checks only in Debug mode.
During release mode, they aren't part of the compiled binary. 
This was a commonplace practice in the old days, however today, many regard being able to assert in production
(or at least enable them from a test window/compiler flag) to be a good thing.
Furthermore, `assert` only takes one parameter. Thus, to get an error message we need to boolean AND our condition with a static string error message. 
Static string constants are `const char *`s, which like integers, are true for nonzero numbers and false for 0 (NULL). 
Truthfully, `assert` is old and generally if you need runtime assertions it's probably better to write your own.

```C++
assert(!myString.empty());

assert(myVec.size() == 2 && "Expected a point");
// "Expected a point" is a static string, which, when converted to an integral value is non-zero
// thus it is considered 'true'
// since true is the null element for AND, it does not affect the test
```

---

> "Programs are meant to be read by humans and only incidentally for computers to execute."
>
> \- Donald Knuth