# Variables

We've already used quite a lot of variables already. So we'll skip the formalities.

### Const Correctness

As I previously mentioned, anything that can be `const` should be `const`. A `const` "variable" is one whose value cannot change. 
It's good practice to make `const` variables the default and mutable variables the exception. 
Mutability makes things harder to reason about and const correctness is another form of type safety.
Like an unsigned vs signed primitive type, a const and non-const type are two fundamentally different types. 
A non-constant can be automatically converted to a constant version of the type, but not the other way around.

```c++
const auto num = 23;
auto mutNum = 32;

num += 10; //error
mutNum += 10; //good
```

### Initialization

In many languages, when you declare a variable, it is set to a default value. 
This isn't the case in C++ since there's no reason for the extra instruction to set a default if the programmer is going to set it to something else anyway. 
It's part of the "pay for only what you use" mentality of C++.
Thus, if you don't initialize a primitive variable to a value, it essentially holds a garbage value. 
What really happens is it will just keep whatever bytes in memory happen to be where the variable now occupies. 
Therefore, **you should always initialize your variables**. This is one of the reasons why it's good practice to 
declare variables to have type `auto`: the compiler needs an initial value to figure out what type the variable is,
so using `auto` will prevent you from forgetting to initialize your variables.

Now you might be asking: "now hold on, you're saying to manually give each of my variables a default value?
So then what's the point of the compiler not doing it?" Well, if you *introduce the variable immediately before you need it (and no earlier)*, 
chances are the initial value you set it to has some bearing on your computation. 
So you don't necessarily initialize a variable to some default value every time,
you initialize it with something that should be based on the logic of your program.

```c++
int a;
std::cout << a << std::endl; // this could print any integer. We don't know what


int powi(int base, unsigned exp) {
    auto res = 1; // initial value of 1 has bearing on computation
    for(auto i = 0u; i < exp; ++i) {
        res *= base;
    }
    return res;
}
``` 

Notice how `res` is introduced right when we need it, and therefore has an initial value which has a meaning for our computation.
Moreover, defining variables as late as possible (and in the smallest scope its needed) makes code easier to read. 

Global and static variables, on the other hand, are initialized to their default value. For most basic data types this is `0`.

### Names

Part of the challenge of programming is giving good names to variables. You want your variables to be self-documenting. Consider the following:

```c++
long timeout; ///< socket timeout in milliseconds
```
vs
```c++
long sockTimeoutMs;
```
or even better:

```c++
std::chrono::milliseconds socketTimeout;
```

Notice we don't need to repeat information encoded by the type.

We also want to be consistent with our variable names. A good rule of thumb is one word per concept. 
For example, it's best not to have some names with `length` and others with `size` when they refer to the same idea such as the size of a container.

### Scope

The variables we have seen have *automatic lifetimes*. This means that when their *scope* ends, they are popped off the stack. 
The stack is an area in memory. Each program is allocated its own stack of limited size. When a variable is declared, it
is pushed on the stack. When a variable goes out of scope, the variable is destroyed and popped off the stack. Having a stack
structure allows variables to be destroyed in the reverse order that they were declared. This allows one variable to depend on another.
We may talk more about this later but for now, know that these variables of built-in types we have been working with are pushed on the stack when 
they are declared and popped off when they go out of scope. 

So what is this scope I talk about?
A scope roughly corresponds to a block, which are delimited by `{}` pairs. 
The local scope is the scope of a function; variables in this scope are destroyed when the function returns. 
The class scope is the scope of class members. Values in this scope are initialized when an instance of the object is created
and destroyed when that same instance is destroyed. 
Finally, the namespace scope is the scope of a namespace. Variables in this scope are destroyed when the program ends.

```c++
{
    auto a = 3;
    // a can be used here


} // a goes out of scope here

// a cannot be used here
```

```C++
class Foo {
    // start class scope
    int a;
public:
    
    void bar() {
        // start function scope
        if (a == 3) {
            // inner scope for if
            int c = a * 3;
        } else {
            // scope for else
            // c cannot be accessed here
        }

        // c cannot be accessed here


    } // end function scope


}; // end class scope


int main() {
    Foo foo; // Foo.a created here
} // Foo is destroyed here, and so is Foo.a
```

### Globals

Global variables are variables that are defined outside any scope (not within a function or curly braces). 
Unless they are immutable, you should avoid globals. In the words of Bjarne Stroustrup: "global variables should have names starting with //". 
This is because the order of variable initialization between *compilation units* (different source files) is undefined.
So if we use a global variable outside the source file it is defined in, then there is the chance we may try to use a variable
that hasn't been created yet or has already been destroyed. Specifically, this occurs if we use a global
in the constructor or destructor of a class with a global instance. A constructor is a function that is called at
the initialization of a variable, and a destructor is called at the destruction of a variable. More on this later.

Globals also make code harder to reason about since it's harder to tell when a function or class depend on them.
This is something we'll talk about more later.

```c++
const auto globalVar = 10;

int main() {
    auto localVar = 0;
    return 0;
}
```

#### Further Reading

[A Tour of C++](https://github.com/Kikou1998/textbook/blob/master/A%20Tour%20of%20C%2B%2B%20(2nd%20Edition)%20(C%2B%2B%20In-Depth%20Series).pdf) 1.4 - 1.6

[Clean Code](https://github.com/ontiyonke/book-1/blob/master/%5BPROGRAMMING%5D%5BClean%20Code%20by%20Robert%20C%20Martin%5D.pdf) Chapter 2 (I **highly** suggest you read this)

[C++ Primer](https://github.com/yanshengjia/cpp-playground/blob/master/cpp-primer/resource/C%2B%2B%20Primer%20(5th%20Edition).pdf) 2.2 - 2.3

