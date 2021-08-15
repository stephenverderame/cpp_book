# Hello World

Let's get coding! Create a file called `main.cpp`. Common file extensions for C++ source code are `.cpp`, `.cc`, and `.cxx`. Here's a Hello World program in C++

```C++
#include <iostream>

int main() {
    std::cout << "Hello World" << std::endl;
    return 0;
}
```

In the first line, we have this `#include` statement. This is a preprocessor directive which includes a *header file* into our current source file. 
We'll talk more about this later, but for now just know that this brings in the declarations of many IO utilities.

Next, we define our insertion point with the `main()` function. The `int` designates the return type as an integer. 
In C++, the main function returns `0` on success, and a non-zero error code on failure. Here we just return `0`. It's perfectly legal to not have the `return` statement in the main method.
In that case the return value is `0`. This is a special property of the `main()` function only. Everything else needs an explicit `return` statement.

`std::cout` is an *output stream* that goes to standard out. Here we print the string `"Hello World"`, and then add the newline character and flush the stream with `std::endl`. 
If you just want a newline, you can just print `"Hello World\n"` directly, which is more efficient if you're doing a lot of printing since it avoids excess flushing. 
`<<` is an operator defined for `ostream`s like `std::cout`. The left-hand argument is a `std::ostream` and the right-hand argument is data to put onto the output stream. We'll talk more about IO later.

To build and run our code. We can invoke g++.

`g++ main.cpp -o hello_world`

This will compile our code and produce an executable called "hello_world". Running it should yield:

> Hello World
