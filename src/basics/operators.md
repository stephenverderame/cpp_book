# Operators

As we will see later, operators can be overloaded to do arbitrary things. However it is good practice to keep operators' semantic meaning. Operators are really just regular functions with a special way to call them.

## Basic Binary Operators

Binary operators operate on two arguments.

* `+`
* `-`
* `/`
* `*`
* `=`
* `+=` - add an amount to a value
* `-=`
* `/=`
* `*=`
* `==` - check for equality
* `!=` - check for inequality
* `<`
* `>`
* `<=`
* `>=`
* `&&` - logical and
* `||` - logical or

It's important to note that `=` operators (`=`, `+=`, etc.) all return the new value that was set. For example:

```C++
auto x = 10;
auto y = 20;

x = y += 30; 
// sets y to 30 + 20 (50) and returns the new value of y (50)
// this value is then assigned to x

auto test_var = 20;

if (test_var = 30) {
    /*
    * This is a common bug to be on the look out for!
    * Here we assing test_var to 30, then the = operator
    * will return the new value of test_var (which is 30)
    *
    * Then the if statement will check if the result of the 
    * guard `test_var = 30` is true. Since this expression
    * evaluates to `30`, which is not 0, the if statement
    * sees the result of the expression as `true`
    * and enters the if-block
    *
    * To test if test_var equals 30, we want 
    * `test_var == 30`
    *
    * This is yet another reason to prefer `const` variables
    * If `test_var` was declared as `const auto`, then the
    * compiler would have given us an error.
    *
    * Most compilers will give a warning about this, so if
    * you compile with the option to treat warnings as errors
    * (which you should), the compiler will catch this
    * even if `test_var` is not a constant
    */
}
```

Here are some other quick examples:
```C++
auto x = 5 / 3; 
// x is 1 (division on integers performs integer division)

auto x2 = 5 / 2.0;
// x2 is 2.5

auto b = 5 * 3 + 2;
// b is 17
b += 10;
// b is 27
b \= 2.0;
// b is an integer
// so this will do double arithmetic, then convert it
// to an int by storing it back in b
```

## Basic Unary Operators

Unary operators operator on one argument

* `!` - logical negation
* `++` - prefix and postfix increment
* `--` - prefix and postfix decrement

The difference between prefix and postfix increment/decrement is 
that the postfix versions increment the variable and return
the old value.

```C++
auto m = 10;
auto n = m++;
// n is 10, m is 11

auto p = ++n;
// p is 11, n is 11
```

## Bitwise Operators

* `|` - bitwise or
* `^` - xor
* `<<` - logical left shift
* `>>` - logical right shift
* `~` - bit negation (unary)
* `&` - bitwise and
* `|=`, `^=`, `<<=`, `>>=`, `&=`

The left and right bitshift operators shift the bits to the left and right respectively, shifting in a 0. 
These operators also have the effect of multiplying or dividing by powers of two.

With the exception of bitwise negation, these operators are all binary 
operators.

The right shift and left shift operators do not take into account the two's complement sign of the number. 
In other words, they always shift in 0 bits in the "empty spaces" that they create.

You may notice that we use `<<` and `>>` for stream in and stream
out. For example (`std::cout << "Hello world\n";`).
This is an example of operator overloading. In C++, `<<` and `>>` 
are commonly used for streams for input (right shift), and output (left shift)

```C++
0b111 << 2;
// 0b11100
// shift 2 bits left
// also multiply by 2^2

0b101 >> 3;
// 0b000
// shift 3 bits right
// also divide by 2^3

~0b101;
//0b010

0b0111 & 1;
// 0b0001

0b101110 ^ 0b010011;
//0b111101

0b1001 | 0b1010;
// 0b1011
```