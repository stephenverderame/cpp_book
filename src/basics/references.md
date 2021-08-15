# References

References are basically aliases for another variable. The reference refers to the data owned by a different variable.

```c++
int number = 5;
int & numRef = number;
numRef = 10;
std::cout << number << std::endl; // 10
```

Although we changed the reference, the variable `number` that it refers to also changed. 
This is because both `number` and `numRef` share the same memory location. We can think of `number` as owning the actual integer data, and `numRef` as owning an address that points to that data.

|`number`|              
|---|                
|5|   


|`numRef`|
|---|
|`&number`|

We can also demonstrate this with `auto` as well. It works the exact same as you'd expect.

```c++
auto data = 0.10;
auto& data2 = data;

data = 3.14;

std::cout << data2 << std::endl; //3.14
```

Since a reference basically just stores the address in memory of the data, then for complex types references prevents excess copying of the data. 
Instead of copying, the reference binds to the object it refers to.

```c++

std::string state = "New York";

auto cpyOfState = state; // copy the string

auto & noCopy = state; // reference
```

References can be `const`. A `const` reference cannot mutate the underlying data it refers to.

```c++
const auto & state2 = state;
state2 = "Hawaii"; // error

state = "Alaska";

std::cout << state2 << "\n"; //Alaska
```

We cannot have references of references and references must always be bound upon initialization.

```c++
int & ref; // error, reference not bound
```

## Dead References

**A reference must have a lifetime that is completely within the lifetime of the object it refers to.** Consider the following:

```c++
std::string & getStr() {
    const auto msg = "Hello " + std::string("World");
    return msg;
}

auto& msg2 = getStr();
std::cout << msg2 << "\n";
```
`msg` goes out of scope, and its memory is freed at the end of `getStr()`. Yet we are returning a reference to `msg` past the end of its lifetime! 
So `msg2` is a *dangling reference* (or *dead reference*) because the data it refers to is invalid. 
So what's going to happen here? We don't know. Our program might terminate with a memory access violation, it might seem to work and occasionally print garbage, or it might set our computer on fire. 
It's *undefined behavior*.

If you're compiling on `-Werror` or `/WX` the compiler will stop you from doing this.

Here's another example: (which won't compile)

```c++
std::vector<std::string&> lotsOfText;

void process() {
    std::ifstream reader("someFile.txt");
    std::string book{
        std::istream_iterator<char>(reader),
        std::istream_iterator<char>()};
    // read entire file into book
    // file could be huge, we don't want to copy that
    lotsOfText.push_back(book);

}

void consume(unsigned id) {
    std::cout << lotsOfText[id] << std::endl; 
}
```

`book` goes out of scope at the end of `process()`. Yet we have no idea when `consume()` will be called. 
By that time, the data referred to by the references in `lotsOfText` will likely have gone out of scope, and `consume()` will access a dangling reference. 
In general (and we'll talk about this later) you don't want containers (such as `std::vector`) to store references. 
In fact, the compiler won't let you. This doesn't mean you're forced to copy the data however, and we'll see some neat features that deal with this in later chapters.

When creating a reference to a variable. The reference must not drop any qualifiers. 
Qualifiers are things such as `const` or `volatile` that qualify the type of the variable. This rule ensures that no const variable has its data mutated out from under it.

```C++
const auto num = 5;
int& num2 = num; // error
const int& num3 = num; //good
```

# Pointers

The `&` symbol when it comes before a name can be read as "address of" and `*` before a name is known as the dereference operator and can be read as "value of". 
The result of the `&` operator is an address to the underlying value. An address is just a number, but it cannot be stored in a regular integral type. 
It must be stored in a pointer. As we saw with references, a pointer does not own the value, it owns a copy of the address which points to the underlying data.

Pointers are arithmetic types, and adding or subtracting from them will add or subtract the size of the type that they point to from the address. 
Unlike references, pointers don't have to be initialized. This is not very safe. At the very least, you should set invalid pointers to `nullptr`, a value reserved for invalid pointers. 
Trying to dereference an invalid/dangling/null pointer is *undefined behavior*. With references, creating such deadly situations is much harder since a reference must always be initialized with a value, 
and most compilers will stop you from taking a reference beyond the lifetime of the object it refers to in some cases.

```C++
int num = 5;
int * num2 = &num;
*num2; //5
*num2 = 10;

num2++; //increment the address stored in num2 by sizeof(int)
//now num2 is a dangling pointer
num2 = nullptr; // mutate the pointer itself, not its data
num; //10

```

A `const` that goes before the `*` in the type declaration of a pointer means that the data pointed to cannot change. 
A `const` after the `*` means that the address the pointer stores cannot change.

```C++
const char * hello = "hello";
*hello = 'g'; //error
hello = "goodbye"; // good

auto num1 = 5;
auto num2 = 10;
int * const num = &num2;
*num = 10; //good
num = &num2; //error


const int * const num3 = &num2;
*num3 = 20; //error
num3 = &num1; //error
```

In the use cases I've demonstrated so far, a reference is preferable to a pointer unless its data truly cannot be known when it is declared. 
Pointers are really remnants from C, and there are safer C++ mechanics to do the job of pointers in most cases.

#### Further Reading

[A Tour of C++](https://github.com/Kikou1998/textbook/blob/master/A%20Tour%20of%20C%2B%2B%20(2nd%20Edition)%20(C%2B%2B%20In-Depth%20Series).pdf) 1.7
