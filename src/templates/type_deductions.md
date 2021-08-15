# Type Deduction

If you have been following the advice of Herb Sutter to almost always use auto, then you will have had much experience with type deduction. 
For the most part, type deduction just works as you'd expect, but there are cases where things might snag you.

```C++
template<typename T>
void fun(ParamType t);

fun(expr);

```

In the following explanation, I will refer to `expr` as the expression passed to the function. `T` as the deduced type of the function, and `ParamType` 
as the adorned type used as the argument for the function. For example:

```C++
template<typename T>
void fun(const T& t);

fun(10);
```

`expr` is `10` which has a type of `int`, `ParamType` is `const T&`, and `T` will be deduced to `int`.

Type Deduction Rules:
* Modifiers specified in `ParamType` are lost by the type deduction of `T`
    * So if `ParamType` is `T&`, any lvalue references passed in to the function will have a deduced type which doesn't include the reference. So passing in `const int&`, `T` will be `const int`. 
    In this example, rvalue references would not be able to be bound.
	* If `ParamType` is `const T&`, then passing in a `const int`, `T` will be deduced to be `int`, not `const int`
* If `expr` is a reference, the reference part is ignored
* If `ParamType` is a universal reference and `expr` is a lvalue then `T` is deduced to be an lvalue references. **This is the only time type deduction deduces a reference**
* If `ParamType` is just `T` (pass-by-value) constness, referenceness, and volatility is ignored.
* If `ParamType` is a reference, and `expr` is an array, then `T` deduces the array's type. It doesn't decay into a pointer like normal. This applies to function pointers as well

`auto` type deduction mostly follows the same rules as above. You can think of `auto` as the `T` in template type deduction. 
One caveat is while template type deduction cannot deduce braces, `auto` type deduction can, and it will deduce as `std::initializer_list`. 
A function that returns `auto` follows template type deduction rules, not `auto` type deduction rules.

The best way to understand this is with examples:

```C++
// When I saw 'T is', I mean to say that 'T is replaced by'
void foo(T t);
void bar(const T& t);
void func(T&&);

std::vector<double> d;
const int c;
volatile bool b;
foo(5); //T is int
foo(c); //T is int

bar(c); //T is int
bar(5); //T is int
bar(b); //T is volatile bool
bar(d); //T is std::vector<double>

func(c); //T is const int&
func(d); //T is std::vector<double>&
func(55); //T is int

bar(std::string("...")); // error, cannot bind to rvalue reference
```

```C++
// Once again when I say "auto becomes", I mean to say that "auto is replaced by" as if
// auto was "T" in the above example

std::vector<double> d;
const int c;
volatile bool b;

auto& f = c; //auto becomes const int
auto&& d2 = d; //auto becomes std::vector<double>&
// auto&& is a universal reference

const auto v = b; //auto becomes bool
const auto& v2 = b; //auto becomes volatile bool
//when I mean "auto becomes" I essentially 
//mean it's as if you typed this:
const volatile bool& v2 = b;

auto initList = {10, 20, 30, 40}; //std::initializer_list<int>
std::vector v = {10, 20, 30, 40}; //std::vector<int>
auto vec = std::vector{10, 20, 30, 40};

auto& v2 = v;
// auto becomes std::vector<int>
// so v2 is an std::vector<int>&
```

The `decltype` rules are very simple. 
It produces the exact type of the expression passed. We can use `decltype` rules in place of `auto` or template rules for variables and return values with the syntax `decltype(auto)`.

```C++
auto operator[](int index) {
    return c[index];
    // template type deduction rules
    // since auto lacks any reference, it is returned by-value
    // so we can't use operator[] to assign values
}

auto& operator[](int index) {
    return c[index];
    // returns lvalue reference
}

auto&& operator[](int index) {
    return c[index];
    // returns lvalue reference
    // auto&& is a universal reference
}

decltype(auto) operator[](int index) {
    return c[index];
    // returns lvalue reference
    // return type of operator[] is an lvalue reference
}


int c = 100;
auto d = c; // auto is int
decltype(c) d = c; // type is int
decltype(auto) e = c; // type is int

int& get(int & i) {
    return i;
}

auto f = get(c); // auto is int
decltype(auto) g = get(c); // type is int&

int i;
int&& f();


auto x3a = i;                  // decltype(x3a) is int
decltype(auto) x3d = i;        // decltype(x3d) is int
auto x4a = (i);                // decltype(x4a) is int
decltype(auto) x4d = (i);      // decltype(x4d) is int&
auto x5a = f();                // decltype(x5a) is int
decltype(auto) x5d = f();      // decltype(x5d) is int&&
auto x6a = { 1, 2 };           // decltype(x6a) is std::initializer_list<int>
decltype(auto) x6d = { 1, 2 }; // error, { 1, 2 } is not an expression (only auto deduces braces to initializer list)
auto *x7a = &i;                // decltype(x7a) is int*
decltype(auto)*x7d = &i;       // error, declared type is not plain decltype(auto)
```

So with this knowledge, let's look at the following example:

```C++
template<typename T>
auto make_unique_cpy(T&& t) {
    using Type = std::remove_reference_t<T>; 
    
    return std::unique_ptr<Type>(new Type(std::forward<T>(t)));
}
```

If a lvalue is passed to `make_unique_simple`, then `T` will be deduced to a lvalue reference. 
Since we can't create a pointer to a reference, we must use `std::remove_reference_t` to ensure that the type being passed to `unique_ptr` and `new` is not a reference.

During type deduction, there may be cases where a reference to a reference is produced. Since such double references are illegal, the compiler follows the rules of *reference collapsing* to produce a single reference. 
This can occur when using `decltype`, type aliases, or during type deduction, for example. When a reference to a reference is produced:
* If either references is a lvalue references, the result is a lvalue reference
* Otherwise, the expression collapses to a rvalue reference.

```C++
template<typename T>
auto func(T&& param) {
    const T&& p2 = param;
    // since const is used
    // p2 is not a universal reference
}

std::string name = "Hello";

func(name);
// T deduced to std::string&
// type of p2 becomes const std::string& &&
// type of p2 collapses to const std::string&

func(10);
// T deduced to be int
// type of p2 becomes const int &&
```