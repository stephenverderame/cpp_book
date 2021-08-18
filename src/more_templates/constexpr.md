# Constexpr

`constexpr` isn't really a template, but serves as a method of doing compile time computations.
`constexpr` denotes that an expression may be evaluated as a compile-time constant expression, provided its dependencies
(function arguments, etc.) are also constant expressions.

A `constexpr` variable must be: a literal type, immediately initialized, and the expression initializing the variable must also be a constant expression.
`constexpr` variables must also have a constant destruction, meaning it is not an array or non-literal class. In C++20, a `constexpr` variable can also be a class
provided it has a `constexpr` destructor.

A `constexpr` function must not be `virtual` and its arguments and return type must be literal types. The function body of a `constexpr` function
must not contain definitions of static or non-literal variables, try blocks, inline assembly, or uninitialized variable definition. The latter three are
no longer required in C++20.

We've discussed what a literal type is before: scalars or primitive types, references, and arrays of literals, just to name a few examples.
However, literal types can also be *literal classes*, which are classes with a trivial destructor. 
Literal classes must also have at least one `constexpr` constructor, be an aggregate type (struct with no user-defined constructor or union),
or be a closure type (lambda). Literal classes must have data members that are also literals.

The `constexpr` constructor of a literal class must satisfy the requirements of a `constexpr` function and initialize every data member. C++20 allows
`constexpr` destructors, however in C++17 the destructor must be compiler-generated.

Literal classes may also have `constexpr` member functions, which are **not** implicitly `const` members (they were in C++11).
So, making a `constexpr` member unable to change the state of the class requires the `const` qualifier just like a normal member function.

```C++
class Rational {
    int num, den;
public:
    constexpr Rational(int numerator, int denominator = 1) :
        num(numerator), den(denominator) {}

    constexpr explcit operator double() const {
        return num / static_cast<double>(den);
    }

    constexpr Rational operator-() const {
        return {-num, den};
    }

    constexpr Rational& operator++() {
        num += den;
        return *this;
    }

    friend constexpr auto operator*(const Rational& a, const Rational& b) {
        return Rational(a.num * b.num, a.den * b.den);
    }
};
```

If an exception is thrown during a constant expression during compile-time evaluation, that exception stops compilation.
The compiler error you'll probably see might be something along the lines of "... failed to result in a constant expression".

