# Strings

A string isn't typically classified as a container, but it's close enough to one to put it in this chapter. We've seen a lot of these so far, so I'll save you some excessive explanations. 

Most of the interface for `std::string` is pretty self-explanatory. 

```C++
std::string name = "Why are all my string variables in this book called name?";

name.front(); // 'W'
name.back(); // '?'

name.at(3); // ' '
// at() performs bounds checking

name[5] = 'k';
// operator[] does not

names.size(); // get length of string

names.insert(0, 'W');

names.push_back('?');
names.pop_back();

names.append("Addendum");
names += "Hello"; // same as append()

names.clear(); // erase

names = "Hello" + std::string(" World");
names.substr(1, 4); // "ell"
// takes start index, end index

names.replace(names.begin(), names.begin() + 2, "Je");
// start iterator, end iterator

names.replace(4, 1, "y"); // "Jelly World"
// starting index, length

if (names.find("Joe") == std::string::npos) {
    // does not contain "Joe"
    // find returns an index or std::string::npos if it is not found
}

const auto startIndex = names.rfind('l'); 
// search for string or char starting from end of string

names.erase(names.begin() + startIndex); // "Jelly Word"
```

In `std::string` and other containers, there is a separate notion of `size` and `capacity`. `size` is the actual amount of elements in the container that are valid, `capacity` is the amount of memory currently allocated for the container including uninitialized memory. `size` can be increased and decreased with the `resize()` method while `capacity` can be increased using the `reserve()` method. The capacity grows at a rate of `2x` the previous capacity, and is increased automatically when the size of the container goes beyond the current capacity. When the container needs to reallocate memory, it has to first, allocate the amount of memory equal to the increase capacity, then move the current elements into this new area of memory and finally delete the old memory. Because of the cost of allocating memory and moving the container, by increasing the capacity by `2x` each time methods like `push_back` and `append` have an *[amortized](http://www.cs.cornell.edu/courses/cs3110/2021sp/textbook/eff/amortized.html)* time complexity of `O(1)`.

`std::string` stores a pointer to a dynamically allocated buffer. The address of this pointer may change for the aforementioned reason. However a `std::string` also typically contains a buffer of 10 to 15 characters which is allocated on the stack. This is known as the small string optimization or SSO and allows strings of a small length to get by without ever making a dynamic memory allocation. Dynamic allocations are very expensive compared to stack allocations so this optimization helps improve performance even though it requires more data to copy during move operations. Thus you may find `sizeof(std::string)` to be quite a bit more than one might expect.

Concatenation with the plus operator creates a new string that is the combination of the two operands. This renders long chains of concatenations pretty inefficient. Luckily we have the `std::stringstream` to help us. We'll discuss streams later, but `std::stringstream` has similar usage as `std::cout`. We can use `operator<<` to append to the stream and then use the `str()` member method to convert everything to a string.

```C++
std::stingstream ss;
ss << "Hello World!" << greeting 
    << "It's " << hour << " o'clock where I am and the weather is "
    << weatherDescription; 

const auto st = ss.str();
```

`std::string` also contains a `c_str()` method to convert itself to a C string. A C string is a pointer to a contiguous buffer of characters ending in the null terminator ('\0'). This special byte serves as a sentinel for functions to know when they reached the end of the string. String literals are all C strings. This is why I've never used the plus operator on two string literals and always converted at least one to an `std::string` first because `operator+(const char*, const char*)` is not a defined function. By the way, you can actually concatenate two string literals by just putting them right next to each other. 

Like function pointers, string literals don't have to be freed since they're never manually allocated. Actually, literals are stored directly in the compiled binary, and the pointer is an address to where that literal is.

```C++
const char * concatLits = "Hello" " World";

std::string concatStr = "Hello" + std::string(" ") + "World";

auto rawStringLiteral = R"DELIM(
    Raw string here
    "" fjedjsd
    new lines part of string too 
    \n will not be a new line but instead the literal character \n
)DELIM";
// DELIM can be whatever you want it to be

const wchar_t * wideStr = L"2 bytes per character";

const auto utf8String = u8"UTF-8";
const auto utf16String = u"UTF-16"; 
// NOT the same thing as a wideStr
// wide string is always 2 bytes per character
// utf-16 is 2 bytes per code point and at least 1 code point per character with at least 1 character per glyph

const auto utf32String = U"UTF-32";
```

Since a C string is not a class and has no `size()` method, we must use a function from the C standard library to get its length: `strlen()`. Unlike `size()`, this function is `O(n)` since it has to keep looking at each byte following the pointer of the string (which points to the first character) until it finds the null terminator which has the value 0.

For non-owning references to strings, C++17 introduces the `std::string_view`. A `string_view` has a very similar interface to `std::string` but one major difference: it does not own its data. This means that the lifetime of `std::string_view` must be totally within the lifetime of whatever string-like object it was constructed from. A `std::string_view` can be constructed from an `std::string`, a C string, a pointer and a length, or a begin and end iterator. The typical implementation of a `string_view` would only have it store a pointer and a size. Thus, copying an `std::string_view` is likely more efficient than moving an `std::string` due to the SSO. But once again, `std::string_view` **cannot** exist past the lifetime of whatever data it was constructed from. C++20 introduces the `std::span` which is to `std::vector` what `std::string_view` is to `std::string`. 

One difference with `std::string_view` is that methods that would normally return a `std::string` such as `substr()` return a non-owning `std::string_view`.

```C++

std::string message = "wee woo wee woo";

std::string_view msgView(message.data(), message.rfind(" wee")); // "wee woo"
// construct from pointer and size
// find returns an index, which is basically the size 0f a substr from start of string
// to the argument passed to find/rfind

std::string_view wee = msgView.substr(0, msgView.find(' '));

wee[1] = 'w';

std::cout << message; // wwe woo wee woo

auto getStr() {
    std::string msg = "Hello";
    return std::string_view(msg);
    // BAD: msg is a local variable and goes out of scope
}

auto msg2 = getStr();
// undefined behavior!
```

Finally, C++ strings are part of the standard template library, so it's not too much of a surprise to realize that `std::string` is an alias for `std::basic_string<char, std::char_traits<char>, std::allocator<char>>`. We'll explore the ramifications of this more later, but for now you should know that this means we can use `std::string` for a string of `char`, `wchar_t` (2 byte characters), `char8_t` (utf-8), `char16_t` (utf-16), and `char32_t` (utf-32). Since the second and third template parameters have default arguments, you can customize the type simply as follows:

```C++
std::basic_string<char8_t> utf8String;
```

However, since `std::basic_string<char8_t>` and `std::string` have different template parameters, they are different types and thus don't fit together as easily.

```C++
auto result = std::string("Hello") + std::basic_string<wchar_t>(L" World"); // error
```

Furthermore, members operate on their type parameter, not on abstract notions defined by multi-byte encodings. So for example, the size of a utf8 string will be how many bytes it is which is not necessarily how many characters it has, using characters in the sense that a human would likely use the term character. 
    