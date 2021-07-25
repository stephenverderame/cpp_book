# Lambdas

You've likely seen lambdas in other programming languages so I'll just get into the C++ specifics.

Lambdas have the syntax:
```
[]() {

}
```
Where `[]` denotes the variables captured by the lambda from the local scope, `()` denotes the arguments, and `{}` denotes the code that is part of the lambda. Data that is captured is copied into the lambda's *closure* which is a runtime object that holds copies or references to captured data and is an instance of the *closure class* that the lambda belongs to. We can think of a lambda as being syntactic sugar for the compiler creating a class that provides an `operator()` and instantiating it like so:

```C++
class ClosureClass {
    int myCapture;
public:
    ClosureClass(int myCapture) : myCapture(myCapture) {};
    // think of lambda capture as constructing a member in the closure class

    bool operator()(int myArgument) {
        return myArgument > myCapture;
    }
};

ClosureClass closure(10);
const auto test = closure(20);

// Lambda version:

int myCapture = 10;
const auto lambda = [myCapture](auto arg) {
    return arg > myCapture;
};

const auto test = lambda(20);
```

Unlike functions, the arguments in lambdas can be `auto`, leaving the compiler to deduce their type using the same method it uses to deduce template types. The return type is also automatically deduced, however it can be manually specified with a trailing return type.

```C++
int capture = 10;
const auto lambda2 = [capture](int arg) -> bool {
    return arg > capture;
};
```

The variables captured by a lambda are, by default, copied unless you manually request a reference by prepending the variable name with `&`.  The captured variables are captured once, where the lambda is defined Although the variables are copied, they are immutable unless you make the lambda a `mutable` lambda, which allows the closure to modify the members of the closure class. 

```C++
int cap = 30;
const auto l2 = [cap]() mutable {
    return --cap;
    // although cap is copied, it cannot be mutated from within the lambda
    // unless you declare the lambda mutable
};

l2();
l2();
const auto r = l2();
// r is 27

cap; // 30

const auto l3 = [&cap]() {
    cap = 20;
    // fine because the closure class member (the reference)
    // isn't changing but the data it refers to is
};

l3();

cap; // 20
```

A better way to capture variables is by *init-capture*. This method allows you to move variables into the closure along with making it very clear how each variable is captured. It uses the syntax `<name> = <expression>` where `<name>` is the name of the variable within the closure that will store the result of `<expression>`. This variable is created by move or copy constructing a new instance of the same type of the result of `<expression>`.

```C++
std::unique_ptr<Person> pp;
int num = 0;
Person p;

const auto l4 = [person1 = std::move(pp), num = num, person2 = &p] {
    num; // copied
    person1; // move constructed from pp
    person2; // pointer to p
};
```

If we wanted a reference instead of a pointer, we can wrap the data in a `std::reference_wrapper<T>` using `std::ref`. A `std::reference_wrapper` provided *value semantics* (is copyable) for a reference. It is implicitly converted to a `T` reference, or explicitly with its member function `get()`. It also provides `operator()` so that if it holds a reference to a callable object, it can directly invoke that callable object without unwrapping it. If we wanted a `const` reference we can create one with `std::cref`. This is needed because passing an lvalue reference will invoke the copy constructor, and rvalue will invoke the move constructor. Therefore, we wrap the reference in a `reference_wrapper`, which can then be copy constructed into the closure.

```C++
const auto l5 = [person3 = std::ref(p)] {
    person3.get().speak();
    const auto name = person3.get().name;
    // person3 is a std::reference_wrapper<Person> not a Person&
    person3.get() = Person();
};
```

`std::reference_wrapper<T>` is also useful to allow containers such as `std::vector` that cannot hold references to be able to. However `std::reference_wrapper`, is just, well a `reference_wrapper` and provides no mechanisms to prevent or check for dangling references. In terms of safety, its not much better than having a container of raw pointers.

```C++
std::vector<std::reference_wrapper<int>> nums;

int myNum = 10;
nums.emplace_back(myNum); // emplace forwards arguments to constructor 
// in this case passing an lvalue reference to constructor
// myNum must live at least as long as its reference held in nums

nums.back() = 20;

myNum; //20
```

Back to lambdas: there are limitations to what can be captured. Static variables cannot be captured and neither can member variables. Static variables can be accessed from within a lambda without capturing them, and member variables can be accessed by copying the pointer of the owning instance object.

Variables in a lambda can be captured by a default capture mode `[&] or [=]`. Default capture modes capture all used variables by reference or by value, respectively. You should prefer init capture or enumerating the variables you capture instead of using default capture modes. The reason is because default captures have some implicit behaviors that can cause you trouble if you're not aware of them. 

Let's start with passing by reference. You must take care to ensure that the object you capture outlives the lambda. Lambdas are very easy to bring out of the scope they were declared in (say, adding it to a vector) and thus anything that is captured must live outside that scope as well.

To avoid this, you might be tempted to just copy all variables you use with the default copy capture mode. But believe it or not, the capture mode `[=]` to pass by value doesn't always have the same semantics you would expect. First and foremost, when you capture member variables you aren't actually copying them. You are copying the pointer to the owning object. Thus the default "pass by value" capture is susceptible to the same dangling pointer problem as passing by reference.

What's more is that although lambda's cannot capture static variables, default capture modes may make it seem like they actually do. One might assume that all variables you use are copied, when in fact they are not. A static variable "captured" by a default capture mode isn't really captured, as the code will simply refer to the single instance of the static variable that exists outside the scope of the lambda. Therefore, a default capture by value isn't actually capturing by value. This confusion can be avoided by using *init-capture*.
```C++
class Foo {
private:
    int data;
    Person person;
//...

    // in some member function:
    // bringing the lambda out of the function scope
    callbacks.push_back([=]() {
        return data * 5;
        // data is not copied
        // the this pointer is
        // if this lambda outlives the owning 
        // instance of Foo
        // we'll have a dangling pointer!
    });
    
    // This is what's really happening:
    callbacks.push_back([ptr = this]() {
        return ptr->data * 5;
    });
    SomeClass c;
    auto p = [=, &c]() {
        // "everything" captured by value except c
    };
    
    // Init capture examples:
    [person = std::move(person), c2 = &c]() {
        // Foo::person is moved to a variable person
        // in the closure
        // c is passed by pointer to a variable c2
        // in the closure
    }
    
}
```
The closure class's `operator()` is by default const. So mutating a captured variable requires declaring your lambda mutable.

# Generic Functions

Lambdas are a type of callable object, but not the only one. In the header `<functional>`, the STL provides a generic function class `std::function` which can wrap any callable object (lambdas, function pointers, objects that define `operator()`, etc.). They are copyable and moveable and otherwise work like you might expect from a function. They enable us to use higher order programming and to pass function-like objects as values. The syntax for declaring one is as follows:
```C++
std::function<ReturnType(Args, ...)>

// example
std::function<void(int, int)>; // takes two integers and returns nothing
std::function<char()>; // returns a char, takes nothing
std::function<int(int, int)>; // returns an int from two ints
```

```C++
using my_func_t = std::function<int(int, int)>;

int sum(int a, int b) {
    return a + b;
}

struct DivOp {
    int operator()(int a, int b) const {
        return a / b;
    }
};

my_func_t f1 = &sum;
my_func_t f2 = DivOp();
my_func_t f3 = [](auto a, auto b) {
    return a * b;
};
```

Here are some more examples:

```C++
struct Test {
    std::string str;

    void print() const {
        std::cout << str << std::endl;
    }
};

std::function<void(const Test&)> func = &Test::print;
Test t = {"Hi"};
func(t); 

std::function<void()> f2 = []() {
    std::cout << "qwertyuiop" << std::endl;
};
f2();
```

The real power of `std::function` is to be able to pass callable objects to functions and classes to control behavior. However, especially in the case of lambdas, you must be careful not to let a function outlive any references it may use.

```C++
/**
* Creates a new container containing only the elements that fell in line
* Order is determined by the function f, which returns true if the first argument should come
*  before the second
*/
template<class Container, typename Element>
auto stalinSort(const Container& container, 
    std::function<bool(const Element&, const Element&)> f) 
    -> std::enable_if_t<std::is_same_v<
        typename std::iterator_traits<decltype(std::declval<Container>().begin())>::value_type,
        Element
    >, Container>
{
    Container result;
    if (container.empty()) return result;
    auto it = container.begin();
    result.push_back(*it);
    it = std::next(it);

    auto last = result.begin();
    for (; it != container.end(); ++it) {
        if (f(*last, *it)) {
            result.push_back(*it);
            // ++last; BAD
            // we cannot just increment last
            // because the container might move 
            // if it needs to reallocate. If this happens
            // last will become invalid
            last = result.begin();
            std::advance(last, result.size() - 1);
        }
    }

    return result;

}

std::vector nums = { 10, 400, 20, 30, 60, 708, -100, 1000 };
auto res = stalinSort<std::vector<int>, int>(nums, [](auto a, auto b) { return a < b; });
// res is 10, 400, 708, 1000
```
## Partial Application and Bind
 
 Generally, lambdas should be preferred to `bind`. But I do feel it has its place. Bind can be used to partially apply a callable object. It takes a callable object, and constructs a *bind object* from it by copying in the arguments passed to `std::bind`. The bind object works similarly to a closure. However it's `operator()` can only delegate to the callable object that was passed to `std::bind`. The first argument of `std::bind` is the callable object, if the callable object is a member function then the second argument is a pointer to the owning instance (or a placeholder). Remember, this owning object pointer can really be viewed as the first argument to the function. The rest of the arguments are placeholders or parameters bound to the underlying function arguments based on position. The arguments passed to `std::bind` are constructed in the bind object. Thus lvalues are copied into the bind object and rvalues are moved. Furthermore, we can use `std::placeholders` to map arguments of the bind object's `operator()` to arguments in the underlying callable object. Each placeholder has a number that corresponds to the argument index during invocation of the bind object. This is best explained with an example:
```C++
class Bar {
public:
    void memberFunction(int a, double & b, char c) {
    }
}
Bar b;
double dub = 0;
auto foo = std::bind(&Bar::memberFunction, &b, 
    5, std::ref(dub), std::placeholders::_1);
// create a bind object for 
// memberFunction belonging to instance b
// to pass a reference, since all variables are 
// copied requires
// wrapping in std::ref()
// all arguments to bind are evaluated at the time 
// of the  call to bind
// the placeholder allows a user to call the bind object 
// with some more arguments

foo('c'); 
// calls b.memberFunction(5, dub, 'c');

char doFoo(std::string name, std::string text);

std::string myStr = "Hello World";
auto foo2 = std::bind(&doFoo, std::placeholders::_1, myStr);
myStr = "Goodbye";

char res = foo2("Joe"); 
// calls doFoo("Joe", "Hello World");

auto foo3 = std::bind(&doFoo, std::placeholders::_2, std::placeholders::_1);
foo3("Hello", "World");
// calls doFoo("World", "Hello")
```
References must be wrapped in `std::reference_wrapper` for the same reason as lambdas. Like lambdas, unless you use a `reference_wrapper` arguments are evaluated at the call site of `std::bind`, not at the call site of the bind object.

```C++
auto str = "Billy";
auto foo4 = std::bind(&doFoo, str, std::placeholders::_1);

str = "Joey";

foo4("Johns");
// calls doFoo("Billy", "Johns")

```