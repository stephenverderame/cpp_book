# Primitive Types

| Type | Min Size |
| ---- | ---- |
| bool | n/a |
| char | 8 bits |
| short | 16 bits |
| int | 16 bits |
| long | 32 bits|
| long long | 64 bits |

In practice, `int` is generally 32 bits, most implementations make it the size of the general purpose registers on your machine, but the standard only stipulates that `sizeof(char) == 1` and `sizeof(char) <= sizeof(int) <= sizeof(long) <= sizeof(long long)`.

Except for bool, these types can be signed or unsigned

```c++

char c; // [-2^7 to 2^7 - 1]
unsigned char c2; // [0 to 2^8]

```

We also have floating point types: `float`, `double` and `long double`. The standard only requires that `double` provides as least as much precision as `float` and that `long double` provides as least as much as `double`. On x86 systems, `float` is typically 32 bits, `double` is typically 64, and `long double` is commonly 80 or 128.

C++ also has literal type specifiers to convert a literal value to a certain type. A literal is something that's already a value such as `42` or `0.0`.

```c++

auto f = 0.1f; // f is a float
auto d = 0.1; // d is double
auto u = 10u; // unsigned int
auto lng = 20l; // long
auto ulng = 20ul; // unsigned long
auto ullng = 200ull; // unsigned long long
auto i = 200; // i is int
auto ch = 'H'; // char

```

Finally we can also input numbers in different bases such as hex, octal or binary.

```c++
auto h = 0xA; //h  is the integer 10
auto oct = 024; //oct is the integer 14
auto bin = 0b11; // bin is the integer 3
```

Hex and binary are pretty important in systems programming so make sure you read up on that if you're unfamiliar.

## Implicit Type Conversions

Implicitly, signed values become unsigned when you mix the two. Thus, we never want to have signed/unsigned mismatches and that's one thing `auto` helps prevent.

```c++
const auto num = 10;
auto n2 = num - 20u; // 2^32 - 10
```

The following type conversions are implicit, where a type can be converted to any type to the right of it:

`char -> short -> int -> long -> long long -> float -> double`.

`bool` can be implicitly converted to any integral type. `true` becomes `1` and `false` becomes `0`. Integral types convert to `bool` where `0` is `false` and everything else is `true`.

If a value goes out of range on an unsigned integral type, the value wraps back around from the lowest point in that type's range. Out of range on signed types is *undefined*.

#### Further Reading

[C++ Primer](https://github.com/yanshengjia/cpp-playground/blob/master/cpp-primer/resource/C%2B%2B%20Primer%20(5th%20Edition).pdf) 2.1 - 2.1.3