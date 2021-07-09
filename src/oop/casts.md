# Casts

With our `Rational` class, we saw how we can convert integers to `Rational` implicitly. What about making a conversion that goes the other way? Can we make `Rational` convert to something like a `double`?

Well yes, yes we can.

```C++
    // in the Rational class
    operator double() const { return num / static_cast<double>(den); }
```

This conversion is implicit. Which can lead to a problem:

```C++
Rational r(10, 2);
if (r == 10) {
    // do we promote 10 to Rational and call operator==(Rational, Rational)
    // or de we convert r to a double and do operator==(double, int)?
}
```

Implicit conversions (either constructor or conversion operator) are not really the best practice. Especially when that conversion is to/from integral types. Consider some things that these implicit conversions allow, but don't make much semantic sense:

```C++
Rational r(10, 20);
if (r == 'h') {
    // convert 'h' to an integer, then to a Rational
}

r = true; 
//convert true to int and int to Rational, then update via operator=
```

Now with a Rational, most things make sense except `char`s and `bool`s. But just like we can make the conversion constructor `explicit`, we can make the conversion operator `explicit`.

```C++
    explicit operator double() const { return num / static_cast<double>(den); }
```

Now we can use `static_cast<double>` to explicitly convert a `Rational`.

```C++
Rational r(10);

const auto rd = static_cast<double>(r);
```

`static_cast<T>(E)` is mostly safe and only permitted if there is a valid conversion from the type of `E` to `T`. Such a conversion may be implicit or explicit. `static_cast` cannot cast away qualifiers such as `const`. `static_cast` can perform upcasts (casting a Derived class reference to a base class reference) and downcasts (casting a base class reference to a derived class reference) provided the cast is unambiguous (no duplication due to diamond hierarchies and multiple inheritance) and non-virtual (no virtual inheritance). However `static_cast` performs no checks that the Base class's actual type is the Derived class we are casting to.

```C++
const auto i = static_cast<int>(54.078);
const auto d = static_cast<float>(true);

class B { 
public:
    virtual ~B() = default; 
};
class D : public B {};

D d;
B & b = d;

auto d2 = static_cast<D&>(b);
// notice that we are casting to a reference to D and not to D itself
// dangerous

auto b2 = static_cast<B&>(d);
// upcast, safe
```

For a safer way to downcast, C++ provides `dynamic_cast<T>(E)`. Like static casts `dynamic_cast` can perform upcasts and addding `const`. But where it differs greatly is during downcasts. `dynamic_cast` only works on polymorphic hierarchies (those which have at least one virtual function). If `T` is a pointer or reference to a type derived from the static type of `E`, and the dynamic type of `E` is `T` or a derived type of `T`, the cast succeeds. Otherwise the cast fails and throws `std::bad_cast` if used on references or returns `nullptr` if used on pointers. `dynamic_cast` works with virtual inheritance, unlike `static_cast` but costs an extra type check at runtime. Furthermore, the cast returns `nullptr` if `E` evaluates to `nullptr` and returns a pointer to the actual type of `E`.

```C++
class B { 
public:
    virtual ~B() = default; 
};
class D : public B {};
class C : public D {};

D d;
B & b = d;
C c;

auto d2 = dynamic_cast<D&>(b);
// notice that we are casting to a reference to D and not to D itself
// now safe, will perform a runtime type check

auto b2 = dynamic_cast<B&>(d);
// upcasts incur no runtime check


auto c2 = dynamic_cast<C&>(b); 
// std::bad_cast
// undefined behavior if this were static_cast
```

## Dangerous Casts

First up on the "bad boy" side of town is `const_cast<T>(E)`. This one allows you to strip qualifiers off the type of `E`.

```C++
const int a = 10;
auto t = static_cast<int>(a); //error

auto t2 = const_cast<int>(a); //works

auto t3 = const_cast<const int>(t2);
```

It also allows you to add qualifiers as well.

`reinterpret_cast<T>(E)` completely reinterprets the bytes of `E` as the bytes of `T`.

```C++
double d = 58.01;
auto d2 = *reinterpret_cast<long long*>(&d);
// d2 is not 58
// instead, it will read the floating point representation of d
// and interpret that as twos-complement

int num = 100;
auto bytes = reinterpret_cast<std::byte*>(&num);
bytes[0]; 
// most likely 100 for little endian, 0 for big endian

const double dub = 10;
auto bytes = reinterpret_cast<std::byte*>(&dub);
// error, cannot cast away const

```

Finally, there is the C-style cast. This guy should be avoided because you will never know what type of cast is actually being performed. The C-style cast has the most power and can pretty much anything to anything else. An example problem: You want to use a conversion function but forgot to implement it. A C-style cast won't complain and will reinterpret the bytes which is **not** what was intended! A `static_cast` would fail to compile.
