# Functions

As we have seen, if functions are not declared and defined separately, we can use `auto` to have the compiler use type deduction on the return type. Otherwise, just like function parameters, the return type must be specified.

```c++
void printHello(); //declaration


int main() {
    printHello();
    return 0;
}

void printHello() {
    //definition
    std::cout << "Hello\n";
}
```

Declaring a function ahead of it's definition like in the above snippet is known as *forward declaring* and it allows us to not have to manually ensure that every function is defined in topological order. It basically tells the compiler "hey, I'm going to define this later so if you see this name before you see its definition don't freak out."

`void` is a special type that essentially means there is no type. We can use it for functions that don't return anything. 

Functions can be *pass-by-value* or *pass-by-reference*. When a function is pass-by-value, the function parameter is copied, when it's pass-by-reference the actual object is not passed to the function, but the address of the object is.

```c++

void add(int a) { // pass by value (copy data)
    a += 5;
    std::cout << a << std::endl;
}

void addRef(int & b) { // pass by reference (bind alias to same piece of data)
    b += 5;
}

int main() {
    auto num = 0;
    add(num);
    // num is still 0 but 5 is printed
    addRef(num);
    // num is now 5
    return 0;
}
```

In `add()` a new copy of `num` is created which is called `a`. `a` goes out of scope when `add()` terminates. In `addRef()`, `b` binds to `num`. Thus, they share the same object and the same data. In `addRef()`, `b` is used as an output parameter because the result of the function is returned in one of its parameters. **You should avoid output parameters.** This is because when you're looking at the call site of `addRef()`, there's nothing to tell you that `num` is being mutated and you may reasonably expect that `num` would retain it's value.

Does that mean we shouldn't pass by reference? No! Far from it. We can pass by `const` reference.

```c++
int addRef2(const int & a) {
    // a += 4; // error, a cannot be mutated
    return a + 5;
}

int main() {
    auto num = 0;
    num = addRef2(num);
    // num is now 5
    // from just looking at this, it's pretty clear num is changing
}
```

In this example, you can look at the call site of `addRef2()` and pretty clearly see that `num` is being mutated. 

For built-in types, passing by reference is likely tantamount to premature pessimization. On a 32 bit OS, and address will be 4 bytes and 64 bit OS will have 8 byte addresses. Therefore, it would be less data being copied if we just passed by value since an address is at best the same size as an integer (in almost all implementations, but not necessarily). However, we will soon see how this is very very useful.

## Function Overloading

Functions with the same name can be overloaded by having different argument types or different amount of arguments. The function with arguments more closely matching the parameters passed is the one that is called.

```c++
void func1(int); //version a
void func1(char); //version b
void func1(double); //version c
void func1(char, int); // version d

short ss = 10;
func1(ss); // version a
func1('H'); // version b
func1(0.1f); // c
func1(true); // a
long l = 100;
func1(l); // a
func1(false, 100ll); // d
```

If there are multiple equally good matches, the compiler won't guess and will not compile. Likewise, if no parameter is implicitly convertible to the type of any overload's arguments, then compilation fails.

## Function Design

> The first rule of functions is that they should be small.
>
> The second rule of functions is that *they should be smaller than that*
[^1]


Making functions small organizes sections of your code into named units. How small is small? Well most should rarely hit 20 lines. For example, when you take/took CS 3110, you'll/you'd lose points if a function is > 20 lines. Furthermore, functions should **do one thing**. How do you know they do one thing? Well you should be able to describe it in about one sentence without using a conjunction like "and."

Example:

```c++
if(person.getAge() >= 18 && person.getAge() < 25 
    && person.getHighestEdu() == EducationLevel::Highschool
    && !person.livingAtHome()) 
{
    // if person attended college after HS
}
```
OR
```c++
inline auto attendCollegeAfterHS(const Person & person) {
    return person.getAge() >= 18 && person.getAge() < 25 
        && person.getHighestEdu() == EducationLevel::Highschool
        && !person.livingAtHome();
}

// ...

if(attendCollegeAfterHS(person)) {

}
```

Notice how by creating a helper function, we were able to encode the comment in a name. Commenting what the code does is unnecessary since the function name says it all. If you find yourself commenting what code does, that's a good hint that you might want to make a function. We also see how `attendCollegeAfterHS()` does just one thing. We also pass by `const` reference since the function only uses accessors of `Person` and doesn't do any mutations.

For another example, we saw in our guessing game how we turned this:

```c++
int main() {
    const auto secretNum = getRandNumBetween(min_value, max_value);
    auto guess = 0;
    do {
        std::cout << "Guess a number between " 
            << min_value << " and " << max_value << std::endl;
        guess = getUserGuess();

        if (guess < secretNum) {
            std::cout << "Too low!" << std::endl;
        } else if (guess > secretNum) {
            std::cout << "Too high!" << std::endl; 
        }
    } while (guess != secretNum);
}
```
into this
```c++
int main() {
    const auto secretNum = getRandNumBetween(min_value, max_value);
    auto guess = 0;
    auto tries = 0;
    do {
        ++tries;
        guess = getUserGuess();
        displayGuessHint(guess, secretNum);
    } while (guess != secretNum);
    displayWin(secretNum, tries);
}
```
Once again, we see that the different tasks done during the main loop are easier to read since they essentially have labelled names. We could be more pedantic and make the loop it's own function as well, but this function is 10 lines long so I felt that was good enough. I like functions to be of a size so that in one "eye-space" I can take in the entire function. So no scrolling, moving my head, etc. Generally I have found this corresponds to 20 lines or less pretty well.

Functions should also not use output parameters, and have a small amount of arguments. Generally shoot for no more than 4. And if you have parameters of the same type next to each other (and order matters), you can encode the order in the function name or separate the parameters by some argument of a different type (if there are more arguments).

Ex.
```c++
void assertExpEqAct(int expected, int actual);
```
vs
```c++
void assertEquals(int expected, int actual);
```
A user of `assertExpEqAct()` wouldn't have to look up the order of arguments in the docs since the order is encoded right into the name.

Here's another example:

```c++
using it = std::vector<char>::iterator;
void copy(it dstBegin, it dstEnd, it srcBegin, it srcEnd);
```
vs

```c++
using it = std::vector<char>::iterator;
struct Range {
    it begin, end;
}

void copyDstFromSrc(Range dst, Range src);
```

Notice how the first function was *missing an abstraction* which led to having 4 parameters. In the second function, we created a `struct` to organize the parameters into an `abstraction`. In C++20, this can be done with `std::span`.

We should also prefer *pure functions* a pure function has no side effects, and returns the same output for the same inputs. It should not mutate variables or have any other effect other than the value it returns.


## Inline Functions

Earlier you saw me use the `inline` keyword. What this does is it *suggests* to the compiler that the function can be inlined. What is an inlined function? Well, we'll cover the details later but basically every time you call a function the state of the current function must be saved, you must jump to the new function, then you must restore the state of the old function and jump back. Abstractly, this process can be viewed as having to push an *activation record* onto the stack and then popping it off. An *activation record* basically contains all the data like arguments being passed, where the function is called from (so it can jump back), and the state of the callee. Sounds like a complex task? Well it sort of is. When a function is inlined, the compiler puts the body of the function right at the call site. So all this jumping and state saving doesn't need to occur. Let's look at another example of factoring out some code into an inline function:

```c++
constexpr auto expFac = 0.83;
const auto experience = person.getAge() * person.getGPA() 
    + expFac * person.getName().size();
```
VS

```c++
inline auto getExperience(const Person & person) {
    constexpr auto expFac = 0.83;
    return person.getAge() * person.getGPA() 
    + expFac * person.getName().size();
}

const auto experience = getExperience(person);
```
For most compilers, the generated machine instructions will be pretty much the exact same. But the second option gives us greater readability.

A function can only be inlined if it is defined and declared in the same place. So an inline function cannot have separate declarations and definitions unless the declaration and definition are in the same file.

## Constexpr Functions

Like `constexpr` variables have values that are available at compile time, `constexpr` function have computations that *can be* available at compile time. If you pass non-constexpr arguments to a `constexpr` function, the function will behave normally, but if you pass literals or `constexpr` variables to a `constexpr` function, the result will be computed at compile time and the literal value will be inserted in the code.

Ex.

```c++
constexpr int fact(int a) {
    if(a <= 1) return 1;
    else return a * fact(a - 1);
}

constexpr auto my_number = 10;

int num = fact(my_number);
// num = 3628800 will be in the compiled binary

int num2 = fact(4);
// num = 24 will be in the binary

int num3 = fact(nonConstexprFunc());
// will behave like a normal function
```

We will explore more about `constexpr` and other ways of performing compile time calculations later. 

#### Further Reading

[A Tour of C++](https://github.com/Kikou1998/textbook/blob/master/A%20Tour%20of%20C%2B%2B%20(2nd%20Edition)%20(C%2B%2B%20In-Depth%20Series).pdf) 1.3

[Clean Code](https://github.com/ontiyonke/book-1/blob/master/%5BPROGRAMMING%5D%5BClean%20Code%20by%20Robert%20C%20Martin%5D.pdf) Chapter 3 (I **highly** suggest you read this)

[C++ Primer](https://github.com/yanshengjia/cpp-playground/blob/master/cpp-primer/resource/C%2B%2B%20Primer%20(5th%20Edition).pdf) Chapter 6

[C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-functions) Functions

---
[^1]: Clean Code p. 34