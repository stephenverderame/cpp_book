# References

References are basically aliases for another variable. The reference refers to the data owned by a different variable.

```c++
int number = 5;
int & numRef = number;
numRef = 10;
std::cout << number << std::endl; // 10
```

Although we changed the reference, the variable `number` that it refers to also changed. We can also demonstrate this with `auto` as well. It works the exact same as you'd expect.

```c++
auto data = 0.10;
auto& data2 = data;

data = 3.14;

std::cout << data2 << std::endl; //3.14
```

You can think of a reference type as storing just the address in memory of the data. Therefore, for complex types references prevents excess copying of the data. Instead of copying, the reference binds to the object it refers to.

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
`msg` goes out of scope, and gets it's memory freed at the end of `getStr()`. Yet we are returning a reference to `msg` past the end of its lifetime! So `msg2` is a *dangling reference* (or *dead reference*) because the data it refers to is invalid. So what's going to happen here? We don't know. Our program might terminate with a memory access violation, it might seem to work and occasionally print garbage, or it might set our computer on fire. It's *undefined behavior*.

Here's another example:

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

`book` goes out of scope at the end of `process()`. Yet we have no idea when `consume()` will be called. By that time, the data referred to by the references in `lotsOfText` will likely have gone out of scope, and `consume()` will try and access a dangling reference. In general (and we'll talk about this later) you don't want containers (such as `std::vector`) to store references. This doesn't mean you're forced to copy the data however, and we'll see some neat features that deal with this in later chapters.

#### Further Reading

[A Tour of C++](https://github.com/Kikou1998/textbook/blob/master/A%20Tour%20of%20C%2B%2B%20(2nd%20Edition)%20(C%2B%2B%20In-Depth%20Series).pdf) 1.7
