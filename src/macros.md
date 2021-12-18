# Macros

We've actually seen macros and the preprocess in action before, but there's actually quite a lot that it can do. But first, let's do a brief overview of the compilation process.

`constexpr` variables and functions should be preferred to macros. But sometimes macros can come in handy for preventing
code duplication in ways no C++ language feature can.

### The Compilation Process

1. <u>Preprocessing</u>

    The first step in the compilation process is preprocessing. I've alluded to this before, but what happens is the preprocessor,
    goes through each source file. It's main functions are to expand out `#include` and `#define` directives
    and handles include guards. The preprocessor doesn't actually know or care about C++ syntax: it operates on the literal text of your source code. 
    The preprocessor is the program that takes header and code files, and basically spits out a bunch of code files.

2. <u>Compilation</u>

    Next is the actual compilation. Each source file is compiled independently as a separate compilation unit. 
    The output of the compiler are a `.obj` files or object files. If a compilation unit uses externally defined symbols 
    (functions, variables, etc that are defined in another compilation unit), the compiler will simply assume all of them are
    valid. The compiler can be broken up into many steps that really aren't important for us to know about.
    However one thing it does is that, when enabled, it optimizes our code. The compiler can do some really cool things
    such as precomputing literal expressions, reordering the machine instructions to promote as much parallelism as possible
    in the CPU pipeline, inline functions, unroll loops, and so much more I have yet to learn about.

3. <u>Linking</u>

    Finally, the linker links together all the object files into a library or executable. The linker is what resolves the
    external symbols in each object file.

All of these 3 steps are handled by three completely different programs. This allows us to swap any of them without regard for the others. Each step in the compilation process doesn't know or care about how the other steps are being caried out.

### Macros

A macro is defined using the `#define` directive. 
When the preprocessor sees a `#define` directive, it will then replace all subsequent usages of the macro with the **text** that the macro has been defined to be.
Remember that the preprocessor happens before compilation, so at this point all of the source code is just that, source code.
We typically use `ALL_CAPS_WITH_UNDERSCORES` to define macros so that they stand out in code and make us wary of their dangers.

```C++
#define MY_NUM 10
#define HIS_NUM 2 + 5

const auto my_var = std::pow(MY_NUM, 4);
// compiler sees: std::pow(10, 4)

const auto the_num = MY_NUM * HIS_NUM;
// compiler sees: 10 * 2 + 5
```

Macros are not stored anywhere. They incur no cost of allocating data.

Notice `the_num` takes on a value of `25`, not `70` which would be the value if `MY_NUM` and `HIS_NUM` were constants or variables.
This is because, the text of `MY_NUM` and `HIS_NUM` are inserted into the code, no computations are done.
The preprocessor has no idea what `2 + 5` even means.
Therefore, it's common to surround macro definitions in parenthesis to ensure they interpretted as a single value (assuming that is the intention).

Another downside of macros is that during debugging, you don't get to see the name of the macro when it is used. The debugger sees what the compiler sees: magic constants.

We can use macros inside other macros. During macro expansion, the result of an expansion is then rescanned and the preprocessor will expand any macros used inside the definition of another macro.

```C++
#define A 1
#define B A + A + 5
#define C B * B

const auto f = C;
// compiler sees: 1 + 1 + 5 * 1 + 1 + 5
```

Macros can take input parameters, which make them look and sort of act like functions. The parameters to a function macro are macros themselves and are expanded just like any other macro.

```C++
#define ADD(X, Y) (X + Y)

const auto b = ADD(100, 300);
//compiler sees (100 + 300)

const auto c = ADD(10, 2) * ADD(2, 4);
// use of parenthesis in the ADD macro so that this eventually becomes 54
// as if this were a function
```

Now what about this:
```C++
#define MUL(X, Y) (X * Y)

MUL(10 + 2, 2)
```

As you might have guessed, this produces the text `(10 + 2 * 2)` and not `12 * 2`.
To prevent this, it's also common to put parenthesis around macro parameters too.

If a macro needs to be multiple lines, we can add a backslash at the end of the line and continue to the next line.

```C++
#define pow(BASE, EXP, RES) {\
    int exp = EXP;\
    RES = 1;\
    while(exp-- > 0) {\
        RES = RES * (BASE);\
    }\
}

auto num = 1;
pow(10, 4, num);
/* Expands to:

{
    int exp = 4;
    num = 1;
    while(exp-- > 0) {
        num = num * (10);
    }
}

*/

```

Now let's talk about times when we might actually find the macro useful. Used inside a macro "function",
`#` will stringify an argument (ie. turn it into a string), no matter what is is. `##` will concatenate two tokens into another token.

```C++
#define LOG(VAR) std::cout << #VAR << "is: " << (VAR) << std::endl

LOG(hello);
// prints to the console "hello is: " followed by the value of the variable hello

#define LASSERT(ASSERT_TYPE, X, Y, MSG)\
    ASSERT_##ASSERT_TYPE(X, Y) << (MSG)

LASSERT(EQ, 5, 4, "Assert test");
// expands to:
// ASSERT_EQ(5, 4) << "Assert test"
// where ASSERT_EQ is another macro
```

Arguments are substituted, then expanded, then we finally rescan for any other macros.

Consider the following:

```C++
#define STRFY(X) #X
#define CONCAT(X, Y) X##Y
#define STRFY2(X) STRFY(X)

STRFY(CONCAT(1, 2));
/*
1. Substitude: 
    #CONCAT(1, 2)
    "CONCAT(1, 2)"
2. Expand: Nothing to expand
3. Rescan: Nothing to rescan



"CONCAT(1, 2)"
*/

STRFY2(CONCAT(1, 2));
/*
1. Substitude:
    STRFY(CONCAT(1, 2))
2. Expand:
    STRFY(1##2)
    STRFY(12)
3. Rescan:
    3a. Substitude:
        #12
        "12"
    3b. Expand: nothing to expand
    3c. Rescan: nothing to rescan


"12"
*/
```

So we see that operating on the result of a macro requires indirection, because an expansion is rescanned for further macros
*last*. This is basically oppposite programming languages which evaluate their arguments *first*.

Macros can also take a variable amount of arguments. These are known as variadic macros. Arguments are separated with commas, and these arguments are "stored" in the macro `__VA_ARGS__` which expands to the literal comma separated list of arguments that were passed into the function.

```C++
#define IVEC(...) std::vector<int>{__VA_ARGS__}

IVEC(20.3, myNum, foo(bar));
//expands to:
// std::vector<int>{20.3, myNum, foo(bar)}
```

Combining this with modern variadic templates can produce pretty powerful stuff. Here's an example of a macro that logs a series
of arguments, and tells us the variable name as well as the value of the variable.

```C++
/**
 * Prints out names[idx] along with f
 * Then recurses by calling log_var_helper and incrementing idx
 * Basically, this unpacks the parameter pack and matches each argument with each
 * member in the names vector
 */
template<int idx, typename First, typename ...Args>
void log_var_helper(const std::vector<std::string>& names, First&& f, Args&&... args) {
	std::cout << names[idx] << ": " << std::forward<First>(f) << "\n";
	if constexpr (sizeof...(Args) > 0)
		log_var_helper<idx + 1>(names, std::forward<Args>(args)...);
}

/**
 * Requires names be a comma separated string, with the same amount
 * of elements as arguments passed
 */ 
template<typename ...Args>
void log_vars(const char* names, Args&&... args) {
	std::vector<std::string> var_names;
	std::stringstream ss(names);
	std::string name;
	while (std::getline(ss, name, ',')) {
		var_names.emplace_back(name.substr(name.find_first_not_of(' ')));
        //split string using the comma as a delimiter
        //  trimming away space in the beginning since the stringified arguments contain space
	}
	log_var_helper<0>(var_names, std::forward<Args>(args)...);

}

#define STRFY(X) #X
#define LOG(...) log_vars(#__VA_ARGS__, __VA_ARGS__)
// could also use STRFY(__VA_ARGS__) as first parameter

int hello = 0;
int goodbye = 10;
auto str = "My name";
std::string b = "Sir";
char f = 'f';
LOG(hello, goodbye, str, b, f, 10 + 2);

/* Prints

hello: 0
goodbye: 10
str: My name
b: Sir
f: f
10 + 2: 12
*/
```