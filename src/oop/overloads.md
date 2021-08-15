# Operator Overloading

Let's first create a class `Rational` to see an example of overloading:

```C++
class Rational {
    int num, den;
public:
    Rational(int numerator, int denominator) : num(numerator), den(denominator) {}

    Rational(int numerator) : Rational(numerator, 1) {}
    // forward call to a different constructor

    Rational(const Rational & other) : num(other.num), den(other.den) {}
    // copy constructor
};

```

This class has two private integers and 3 constructors. 
The first is a constructor that passes in the values for `num` and `den`. 
The second is a conversion constructor from `int`. 
You could also have given `denominator` a default argument like so: `Rational(int numerator, int denominator = 1)`. 
Both cases allow for implicit conversion to `Rational` from `int` (and types that implicitly convert to `int`) since we don't use the `explicit` keyword. 
The third is a special kind of constructor known as a `copy` constructor. We'll talk more about this later, but basically it allows us to create a new `Rational` by copying (and not modifying) an existing one.

```C++
Rational r = 5; // invoke constructor 2
auto r2 = r; // copy constructor
Rational r3(10); // ctor 2
Rational r4{20}; // ctor 2
Rational r5(10, 2); // ctor 1
Rational r6{20, 10}; //ctor 1
```

Constructors can use `()` or `{}`. This is to avoid *the most vexing parse* which we'll discuss later.

We have already seen how to overload functions, but we can also overload operators as well.

Binary operator overloads take two arguments (the left and right operand), and unary operators take one. 
If the two arguments are different types, then you'd have to define two overloads in order for the operator to be commutative. 
One overload has type `x` as the left operand (first arguments) and type `y` as the right (second arguments) and the other is the opposite with `y` for the left operand (first arg) and `x` as the right.

But first, it's important to realize that methods (member functions) of a class have an implicit first argument that is the context object for that function. 
So if you define a binary operator overload as a member function, the left argument will be an instance of the class.

Let's define the assignment operator to allow updating our object.

```C++

    //...

    Rational& operator=(int num) {
        this->num = num;
        den = 1;
        return *this;
    }

    Rational& operator=(const Rational & num) {
        this->num = num.num;
        den = num.den;
        return *this;
    }
};
```

```C++
Rational r = 10;
r = 20; // assignment op 1
const Rational r2(5, 2);
r = r2; //assign 2
```

All methods of a class have a `this` pointer, which refers to the calling context of the method. 
This calling context is the implicit first argument of member methods. Therefore, `operator=` is a binary operator despite its overload appearing to only have one argument. 
Moreover, `operator=` typically returns a reference to the updated object; so we return `*this`, which dereferences the `this` pointer. 
Once again notice how `num` in the second overload is `const &`. It is `const` since there is no need to mutate it, and we pass by reference to avoid extra copying (plus this is the idiomatic way of defining `operator=`). 
You may also notice the strange `->` operator. This is the pointer scope resolution operator, basically the dot operator for pointers. 

Let's define basic arithmetic operations. 
I typically like to define arithmetic operators as free functions because it's slightly more clear what the type of the left operand is (otherwise, the left operand is the implicit first argument `this`). 
Let's look at both ways:

```C++
    // ...
    // member method
    Rational operator+(const Rational & other) const {
        return {num * other.den + other.num * den, den * other.den};
        // No need to specify "Rational {...}" because the compiler can see
        // that this function returns a Rational
    }

};

//free function
Rational operator*(const Rational & a, const Rational & b) {
    return { a.num * b.num, a.den * b.den };
}

```

Since we are creating a new `Rational` we declare the member function to be `const`, this way it takes `const this` as its implicit first argument.

Now you might be curious: "Aren't `num` and `den` private members? How can we access them outside the class `Rational`?" 
The answer is we can't. Well, not without declaring the function a `friend`. 
`friend` classes and functions are classes and functions do not have to be defined in the class scope, but are essentially part of the class they are friends with. 
They have access to all private, protected, and public members. 
They should be used sparingly, as it is the strongest coupling relation available. In this case, it's a good choice since we want `operator*` to behave like a member of the class itself.

Now as currently written, you would expect addition to be commutative, however:

```C++
Rational r(5, 3);

auto r2 = r + 10; // good
r2 = 10 + r; // error
```

It's not! That's because, as defined, `operator+` expects its first argument to be a Rational object and member functions will not do implicit conversions on the implicit first argument. 
However, in the third line we pass an integer. Therefore, we'll need to define a free function which has `int` as the left-hand argument.

```C++
Rational operator+(int a, const Rational & b) {/*...*/}


Rational operator+(const Rational & a, int b) {/*...*/} 
// equivalent to the operator+ we just defined as a member
```

Or, we can just define one free function which takes `Rational` since `int` can be implicitly converted to `Rational`.

Let's also make `Rational` able to be printed to `cout`. For that we can overload `operator<<`, which takes a reference to an `std::ostream`, a super type of the class that `std::cout` is an instance of.

```C++
std::ostream& operator<<(std::ostream & stream, const Rational & r) {
    stream << r.num << "/" r.den;
    return stream;
}
```

Two final overloads I want to give special attention to are the increment/decrement operators. 
Both of these have a postfix and prefix version which do different things. 
The prefix version directly increments the object while the postfix version makes a copy, increments the object, and returns the copy made. 
Therefore, unless you need the old value, it's good practice to use the prefix increment/decrement by default. 
Furthermore, to make compiler optimizations easier, it's smart to implement the postfix operators in terms of their prefix counterparts (it's good code reuse as well).

```C++
    //...
    Rational& operator++() {
        num += den;
        return *this;
    }

    //postfix increment
    Rational operator++(int) {
        const auto cpy = *this;
        ++(*this);
        return cpy;
    }
};
```

Here's a completed Rational class:

```C++
#include <numeric>
#include <ostream>

class Rational {
    // Invariant: num and den are in simplest form
    int num, den;
    friend Rational operator+(const Rational&, const Rational&);
    friend Rational operator*(const Rational&, const Rational&);
    friend Rational operator/(const Rational&, const Rational&);
    friend Rational operator-(const Rational&, const Rational&);
    friend bool operator==(const Rational&, const Rational&);
    friend bool operator<=(const Rational&, const Rational&);
    friend bool operator>=(const Rational&, const Rational&);
    friend bool operator<(const Rational&, const Rational&);
    friend bool operator>(const Rational&, const Rational&);
    friend bool operator!=(const Rational&, const Rational&);
    friend std::ostream& operator<<(std::ostream&, const Rational&);
public:
    Rational(int numerator, int denominator) : num(numerator / std::gcd(numerator, denominator)),
        den(denominator / std::gcd(numerator, denominator)) {}

    Rational(int numerator) : Rational(numerator, 1) {}
    // forward call to a different constructor

    Rational(const Rational& other) : num(other.num), den(other.den) {}
    // copy constructor

    Rational& operator=(int num) {
        this->num = num;
        den = 1;
        return *this;
    }

    Rational& operator=(const Rational& num) {
        this->num = num.num;
        den = num.den;
        return *this;
    }

    Rational& operator++() {
        num += den;
        return *this;
    }

    //postfix increment
    Rational operator++(int) {
        const auto cpy = *this;
        ++(*this);
        return cpy;
    }

    Rational& operator--() {
        num -= den;
        return *this;
    }

    //postfix decrement
    Rational operator--(int) {
        const auto cpy = *this;
        --(*this);
        return cpy;
    }

    


};

Rational operator+(const Rational& a, const Rational& b) {
    return { a.num * b.den + b.num * a.den, a.den * b.den };
}

Rational operator-(const Rational& a, const Rational& b) {
    return { a.num * b.den - b.num * a.den, a.den * b.den };
}

Rational operator/(const Rational& a, const Rational& b) {
    return a * Rational { b.den, b.num };

}

Rational operator*(const Rational& a, const Rational& b) {
    return { a.num * b.num, a.den * b.den };
}

bool operator==(const Rational& a, const Rational& b) {
    return a.num == b.num && a.den == b.den;
}

bool operator!=(const Rational& a, const Rational& b) {
    return !(a == b);
}

bool operator<=(const Rational& a, const Rational& b) {
    return a < b || a == b;
}

bool operator>=(const Rational& a, const Rational& b) {
    return a > b || a == b;
}

bool operator<(const Rational& a, const Rational& b) {
    return a.den > b.den || (a.den == b.den && a.num < b.num);
}

bool operator>(const Rational& a, const Rational& b) {
    return !(a < b) && a != b;
}

std::ostream& operator<<(std::ostream& stream, const Rational& r)
{
    stream << r.num << "/" << r.den;
    return stream;
}
```

In C++20, we can let the compiler generate all those comparison functions for us by just defining a single function: `operator<=>`.

---

### Possible Exercises

1. Create a Vec3d class (or whatever you'd like to call it) which stores 3 doubles and represents a 3D vector in the Cartesian plane. It should support the following operations:
    * `operator+` and `operator-` (for vectors and scalars)
    * `operator*` and `operator/` for scalars
    * `operator*` for vectors which will be the dot product
    * `operator<<` and `operator>>`
    * `operator+=`, `operator-=` for vectors and scalars
    * `operator[]` where index 0 gets the `x` value, index 1 `y`, and 2 `z`.
        * Out of bounds is undefined behavior and you can do (or not do) whatever you see fit
    * It's data may or may not be encapsulated
    * Separate the interface and implementation into a header and code file
        * Can you do this without any include directives in the header file?
