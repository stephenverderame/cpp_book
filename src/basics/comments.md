# \# Headers
(pun intended)

Header files are a way to separate an interface from an implementation and declarations from definitions. We'll discuss how the compiler works in more detail later. 

We have already seen header files with the `#include` preprocessor directive. Header files contain the *declarations* of classes, structs, enums, and functions and in some cases a few definitions as well. Remember how the compiler needs to know the definition of a function before it's used unless that function is *forward declared*? Well this same principle applies to other things like classes and structs and header files are a way to put all the declarations in one place and use these declarations in all files that need them by including the header file.

This separation is also a great way to provide separate client and implementor views of a module. A client need only look at the declarations and relevant comments in a header file, while an implementor would work with the source code in the source file.

Furthermore, `#include` directives are a good way to identify the dependencies of a module.

Header files typically end in `.h` or `.hpp`. Some header files in the standard library allow the `.h` to be omitted when including them as you saw with directives such as `#include <iostream>`.

Let's take a look at an example. Let's create some basic math functions:

my_math.h
```c++
#pragma once
#ifndef MY_MATH_H
#define MY_MATH_H

/**
* @return base^{exp}
*/
int powi(int base, int exp);

/**
* @return base^{exp} % mod
*/
int powMod(int base, int exp, int mod);
#endif
```

`#ifndef` is a preprocessor director standing for "if not defined". So if the macro `MY_MATH_H` is not defined, then we define it and declare `powi()` and `powMod()`. This is known as an *include guard*. The reason we need this is because we might include a header file multiple times. For example: perhaps another header `your_math.h` requires one of these definitions. Now a client might make the mistake of including both `your_math.h` and `my_math.h`. The client shouldn't have to know that `my_math.h` is already included in `your_math.h`, but without the include guards they would. This is because the preprocessor basically replaces an include directive with the contents of the file that's being included. Thus, without include guards, declarations can be repeated in the same file which will cause a compilation error. We cannot have multiple declarations with the same name and parameters in the same file; include guards prevent this. The first time a header is seen a macro (in this case `MY_MATH_H`) is defined. The next time that header is included the macro is already defined, so the `#ifndef` directive will prevent re-declaring the functions.

`#pragma once` is a non-standard way to do the exact same thing. All the major compilers (g++, MSVC, clang) support `#pragma once`, but it's perfectly legal for a compiler not to. For portability, use include guards. However `#pragma once` may be more efficient, and a compiler that doesn't support it won't complain about it so for maximal benefit put `#pragma once` on the first line of a header, and then include guards.

Now we need to define it:

my_math.cpp
```c++
#include <my_math.h>

int powi(int base, int exp) {
    // ...
}

int powMod(int base, int exp, int mod) {
    // ...
}
```

To use these functions, say in main.cpp we just need to include the header file.

main.cpp
```c++
#include <my_math.h>
#include <iostream>

int main() {
    std::cout << powMod(10, 13, 99);
}
```

There are a few cases where definitions can (and must) be in the header file. An example would be a function declared `inline`. However, normal function definitions *must* be in a separate file. This is to prevent duplicate definitions of a function from getting into the binary. If this happens the *linker* will be unable to know which definition to use. How might this happen? Well consider two independent files `file1.cpp` and `file2.cpp`. Neither depend on each other and both use `powi()` so they need to include `my_math.h`. If the definition for `powi()` was in the header, then we would have just introduced duplicate functions with the same name and parameters (so they can't be overloads). This is way definitions are in source files and declarations are in header files. Both `file1.cpp` and `file2.cpp` need forward declarations for `powi()`. Instead of having multiple declarations (which in this case is allowed since they don't depend on each other) we put them in a header file to promote code reuse.

Avoid cyclic dependencies like the plague.

# Comments

```c++
// single line comment

/*
    Multi line comment
*/
```

We'll soon use Doxygen to turn comments into HTML documentation. But for now, I want to point out a few small things.
* We should avoid duplication in comments. This includes stating the obvious and literally repeating yourself
* Comments should be short and sweet
* When you write a comment, think about how you might be able to convey the information in the language itself (for example, a better variable name). If you cannot, then comment.
* Use comments to explain a decision, and communicate information to other programmers like exception guarantees, side effects, etc.

For documentation, you can use Javadocs style

```c++
/**
* Desc
* @param
* @return
* @throw
* @author
*
*/
```

There's a lot to say about this, but others have already said it better...

### Further Reading

[A Tour of C++](https://github.com/Kikou1998/textbook/blob/master/A%20Tour%20of%20C%2B%2B%20(2nd%20Edition)%20(C%2B%2B%20In-Depth%20Series).pdf) 3.2 Separate Compilation

[Clean Code](https://github.com/ontiyonke/book-1/blob/master/%5BPROGRAMMING%5D%5BClean%20Code%20by%20Robert%20C%20Martin%5D.pdf) Chapter 4 Comments

[Pragmatic Programmer](https://www.cin.ufpe.br/~cavmj/104The%20Pragmatic%20Programmer,%20From%20Journeyman%20To%20Master%20-%20Andrew%20Hunt,%20David%20Thomas%20-%20Addison%20Wesley%20-%201999.pdf) The Evils of Duplication and It's All Writing 