# \# Headers
(pun intended)

Header files are a way to separate an interface from an implementation and declarations from definitions. We'll discuss how the compiler works in more detail later. 

We have already seen header files with the `#include` preprocessor directive. The preprocessor is a program that runs before the compiler that handles
directives that begin with `#`. The preprocessor manipulates the text of the source code directly.

Header files contain the *declarations* of classes, structs, enums, and functions, and in some cases a few definitions as well.
A declaration basically tells the compiler that some class, struct, etc. exists, but doesn't tell the compiler exactly what that thing is.
```C++
struct MyStruct; // declaration of MyStruct

// definition of MyStruct
struct MyStruct {
   int foo(); // declaration of foo
};

int MyStruct::foo() {
    // definition of foo
    return 0;
}
```
Remember how the compiler needs to know the definition of a function before it's used, unless it's *forward declared*?
Well a header file essentially provides the forward declarations for a bunch of functions and objects so that we only need to provide the definitions
in one source file. Header files are a way to put all the declarations in one place and use these declarations in all files that need them.
Files that need the declaration can include them by including the header file.

Every source file, and all header files it includes make up a single translational unit.
Each translational unit is compiled independently.
A single translational unit may only contain one declaration for each function, class, etc.
A single program may only contain one definition for each object.
The purpose of the header is to provide the declarations of objects defined one translational unit to other translational units.

The header/source separation is also a great way to provide separate client and implementor views of a module. 
A client need only look at the declarations and relevant comments in a header file, while an implementor would work with the source code in the source file.

Furthermore, `#include` directives are a good way to identify the dependencies of a module.

Header files typically end in `.h` or `.hpp`.
Some header files in the standard library allow the `.h` to be omitted when including them as you saw with directives such as `#include <iostream>`.

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

`#ifndef` is a preprocessor director standing for "if not defined". 
So if the macro `MY_MATH_H` is not defined, then we define it and declare `powi()` and `powMod()`. This is known as an *include guard*. 
The reason we need this is to prevent including a header file multiple times. 
For example: perhaps another header `your_math.h` requires one of these definitions, so it includes `my_math.h`.
Now a client might make the mistake of including both `your_math.h` and `my_math.h` separately.
The client shouldn't have to know that `my_math.h` is already included in `your_math.h`, but without include guards they would.
This is because the preprocessor basically replaces include directives with the contents of the file that's being included.
Thus, without include guards, declarations can be repeated in the same file, which will cause a compilation error.
In this example, without include guards `powi` would be declared once when `your_math.h` includes `my_math.h`,
and a second time when the user includes `my_math.h`.
We cannot have multiple declarations with the same name and parameters in the same file; include guards prevent this.
The first time the preprocessor reads a header, a macro (in this case `MY_MATH_H`) is defined.
The next time it's read, the `#ifndef` (if not defined) directive will prevent re-declaring the functions.

`#pragma once` is a non-standard way to do the exact same thing.
All the major compilers (g++, MSVC, clang) support `#pragma once`, but it's perfectly legal for a compiler not to. 
For portability, use include guards. However `#pragma once` may be more efficient, and a compiler that doesn't support it won't complain about it; 
for maximal benefit put `#pragma once` on the first line of a header, and then include guards.
(Now do I use include guards? Hardly; I haven't ran into a compiler where `#pragma once` wasn't supported)

Now we need to define the functions:

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

There are a few cases where definitions can (and must) be in the header file.
An example would be a function declared `inline`.
However, normal function definitions *must* be in a separate file.
This is to prevent duplicate definitions of a function from getting into the binary.
If this happens, the *linker* will be unable to know which definition to use.
The linker is a program that is executed after the compiler which links object definitions to where they are used.

So how are duplicate definitions introduced?
Well consider two independent files `file1.cpp` and `file2.cpp`.
Neither depend on each other and both use `powi()` so they both need to include `my_math.h`.
If the definition for `powi()` was in the header, then we would have just introduced duplicate functions with the same name and parameters
(so they can't be overloads).
This is why definitions are in source files and declarations are in header files.
Both `file1.cpp` and `file2.cpp` need forward declarations for `powi()`.
Instead of having multiple declarations in each source file, we put them in a header file to promote code reuse.

Let's go back to the `my_math` example.
First the preprocessor runs to put the content of the included header files into `main.cpp` and `my_math.cpp`.
It replaces the `#include` with whatever the included file contains.
Then the compiler compilers `my_math` and `main` into object files.
The linker will then link together the definitions in `my_math.cpp` with the declarations in `main.cpp`.
Without the declarations, the compiler would fail because it wouldn't know what `powi` and `powMod` were; they would be undeclared names.
If we provided the definitions and the declarations in the header file, then the linker would fail because it would find
multiple definitions of the same thing in the program, and not know which one to use.
We could, however, scrap the header file entirely and declare and define `powi` and `powMod` in each source file.
In this case, the functions would be compiled completely independently, and the linker would not try to link the definitions
because they were not declared in a header file.
The obvious problem with this idea is that we'll end up duplicating a lot of code.

Avoid cyclic dependencies like the plague.

# Comments

```c++
// single line comment

/*
    Multi line comment
*/
```

We'll soon use Doxygen to turn comments into HTML documentation. But for now, I want to point out a few small things.
* We should avoid duplication in comments. This includes stating the obvious and literally repeating yourself.
* Comments should be short and sweet
* When you write a comment, think about how you might be able to convey the information in the language itself (for example, a better variable name). If you cannot, then comment.
* Use comments to explain a decision, and communicate information to other programmers like exception guarantees, side effects, etc.

For writing documentation, you can use Javadocs style

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