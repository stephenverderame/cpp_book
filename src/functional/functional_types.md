# Tuple

Like `std::array`, `std::tuple` is a fixed size container, but the difference is it can hold different types. A tuple can be created by enumerating the values in its constructor or using one of the following:

* `std::tie()`
    * Creates a tuple of lvalue references
* `std::make_tuple()`
    * General usage way to make a tuple. Same idea as `std::bind`, arguments are copy or move constructed
* `std::forward_as_tuple()`
    * Creates a tuple of references, lvalue references if the argument is an lvalue, rvalue references if its an rvalue.

Accessing a tuple uses `std::get`, which takes either a type or index as the template argument. Therefore, you can only use `std::get` if you know the type or index you want to get at compile time. We can use the template `std::tuple_size_v<T>` to get the size of a tuple and we can use `std::tuple_element_t<I, Tuple>` to get the type of index `I` of tuple `Tuple`.

```C++
std::tuple<int, char, double> myTuple{5, 'c', 83.0};

auto tuple2 = std::make_tuple(23, true, 'a', 5.f);
float f = std::get<3>(tuple2); // gets value at index 3, which is 5.f
bool b = std::get<bool>(tuple2); // gets the first bool in the tuple, which is true

constexpr auto sz = tuple_size_v<decltype(tuple2)>; // 4

using type_2 = tuple_element_t<decltype(tuple), 2>; // double

std::tuple<float&, bool&> t3 = std::tie(f, b);
// tuple holds lvalue references!
// what it refers to must outlive it

std::tuple<bool&, int&&, double&&> t4 = std::forward_as_tuple(b, getInt(), 5.374);
auto c = std::get<0>(t4);
c = false;
 
```

What if we need to get a value from a tuple from an index computed at runtime? We can recursively call a template function that has the index as a template argument.

```C++
template<class Tuple, size_t N = 0>
constexpr auto dynamicGet(const Tuple& tuple, size_t index) 
    -> std::enable_if_t<N < std::tuple_size_v<Tuple>, std::tuple_element_t<Tuple, N>> 
{
    if (index == N)
        return std::get<N>(tuple);
    else
        return dynamicGet<Tuple, N + 1>(tuple, index);
}

const auto t = std::make_tuple(10, 20);
const auto r = dynamicGet(t, 1);
```
Using this function requires the compiler to instantiate a new `dynamicGet` for the specific tuple type for all indices of the tuple. How this works is that we first call `dynamicGet<Tuple, 0>`. If the runtime index `index` is equal to the compile time index of the function (`N`), then we call `std::get<N>`. Otherwise we call the next function.

Tuples are copyable and moveable if all of their contents are copyable and moveable, respectively. Tuples cannot be modified. The only way to do this would be to swap two tuples of the same type. Furthermore, we can make larger tuples by concatenating existing tuples together with `std::tuple_cat`.
```C++
std::tuple<int, double, char> t = std::make_tuple(5, 5.0, 'c');
std::tuple<int, double, char> t2;
// default construct able if each type it 
// holds is as well
t2.swap(t);
std::swap(t, t2);

std::tuple<int, double, char, float, bool> t3 = std::tuple_cat(t2, {500.f, false});
std::get<3>; // 500.f
```
The main usage of tuples is for passing or returning multiple things from a function. Returning a tuple should be preferred to output parameters. However accessing the data via `std::get` is quite annoying. So we can use a *structured binding* to unpack a tuple. A structured binding unpacks a `std::tuple`, `std::array`, or `struct`. Each member of a tuple is assigned to the corresponding name specified within the `[]` of the structured binding. The general syntax is as follows:
```C++
auto [name_1, name_2, ..., name_n] = std::tuple<type_1, type_2, ..., type_n>();
```
The nth value in the tuple is assigned to the nth name in the structured binding. A structured binding must have the same number of names as values in the object being unpacked. If `auto` (or any form of `auto`) is used, the type of each name is deduced from the unpacking. If you manually specify the type, then all names must have the same type.

```C++
struct Bar {
    std::string foo;
    unsigned fizz : 5;
    unsigned buzz : 27;
};

auto [foo, fizz, buzz] = Bar{"Hello", 30, 120};
// unpacks a struct, left most name in binding corresponds to top left most member in struct definition

std::array myArray = {100, 200, 300};
int [a, b, c] = myArray;
// a = 100, b = 200, c = 300
```

Structured bindings make using tuples to return multiple things really easy:

```C++
auto myFunc() {
    Person p;
    return std::make_tuple(p, 10);
    // person is copied
}

template<typename T>
auto getForward(T&& t) {
    return std::forward_as_tuple(t, 5);
    // 5 is moved
    // t is forwarded
}

auto [person, count] = myFunc();
// person has type Person
// count has type int

const Person p;
auto&& [p3, num] = getForward(p);
// p3 has type const Person&
// num has type int&&
```

Another usage of tuples is to use them to pass arguments to functions via `std::apply`. As its first parameter, it takes a callable object and as a second parameter it takes a tuple which will be unpacked as the arguments for that callable object.

```C++
auto f = [](int a, int b){
    return a + b;
};
auto tup = std::make_tuple(50, 23);
auto res = std::apply(f, tup);
//res is 73
```

# Optional

An optional is a type that may contain a value. It's especially useful for returning something from a function that can fail. If an optional contains a value, that value is part of the optional's memory layout. That is to say the value it contains is not dynamically allocated. If the optional does not contain a value, that value has yet to be constructed and there is no memory reserved for. Therefore, an optional provides practically no overhead to code that uses them regardless if they are empty or not.

An optional's `has_value()` member can be used to query if the optional contains a value, the optional is also convertible to `bool`. It has a default constructor that creates an empty optional and a conversion construct from the type it holds to create a contained optional. We can use `operator*`, `operator->`, or the `value()` members to get access to the contained value of the optional. If the optional does not contain a value when one of these members are called, it throws `std::bad_optional_access`. Remember that dereferences a `nullptr` is undefined behavior. So an optional provides a defined result when it is empty.

```C++
auto opt = std::make_optional(100);
if (opt) {
    std::cout << *opt << std::endl;
    // prints 100
}
struct Data {
    int32_t number;
    uint64_t id;
    int16_t fs[100];
};

std::vector<Data> dataFrames;

std::optional<Data> readFile(std::string_view s) {
    std::ifstream stream(s.data(), std::ios::binary);
    if (stream.is_open()) {
        Data d;
        stream.read(reinterpret_cast<char*>(&d), sizeof(Data));
        return d;
    }
    return {}; // default construct, empty optional
}

if (auto result = readFile("data.bin"); result) {
    // This is called init-if
    // the first statement is the initializer
    // the second is the condition that is tested

    const auto t = result->number;
    dataFrames.push_back(result.value());
}
```

We can construct an element directly in the containing optional with the `emplace()` member function. Furthermore, we can use `value_or()` to get a default value if one doesn't exist, and use `std::nullopt` to indicate an empty optional.

```C++
std::optional<int> p = std::nullopt; // empty
int v = p.value_or(-1); // p is -1

p.emplace(100);
if (p != std::nullopt) {
    std::cout << p.value() << std::endl; // prints 100
    p.reset(); // makes it empty
}
```

Optionals are a great tool for error handling, especially when the error might be somewhat expected every now and again such as a user typing in an incorrect path and the system not being able to open the specified file.

# Variant

The Variant is a type safe union. It can hold one of the multiple types specified as template arguments. A possible usage of a variant is to return either an error message, or a result from a function. Like an optional, the variant's data is held directly within the variant so no dynamic allocation. Furthermore, it does not hold extra data besides the value that is currently stored within it. We can use `std::get` to try and get the specified type or index of the variant. If the variant does indeed hold that type, the value is returned. Otherwise the exception `std::bad_variant_access` is thrown. Once again this requires us to know the type or index of the type we want at compile time. We can use the `std::holds_alternative<T>` function to check if the variant holds a value of type `T`.

```C++

std::variant<int, char, std::string> var = 'H';

if (std::holds_alternative<std::string>(var)) {

} else if (std::holds_alternative<char>(var)) {
    std::cout << std::get<char>(var) << std::endl; // prints 'H'
}
```

There is also the `std::get_if<T>` function takes a pointer to a variant and returns a pointer to `T` if the variant contains `T` otherwise `nullptr`.

```C++
std::variant<Data, std::string> readData(std::string_view s) {
    std::ifstream stream(s.data(), std::ios::binary);
    if (stream.is_open()) {
        try {
            Data d;
            stream.read(reinterpret_cast<char*>(&d), sizeof(Data));
            return d;
        } catch (const std::exception & e) {
            return e.what();
        } catch (...) {
            return "An unknown error occurred while reading file";
        }
    }
    return "Could not open file";
}

auto v = readData("data.bin");
if (auto ptr = std::get_if<Data>(&v); ptr) {
    Data cpy = *ptr;
    // ...
} else {
    std::cerr << "An error has ocurred: " << std::get<std::string>(v) << std::endl;
}
```


Another way to determine what a variant holds besides `std::get_if` is the member function `index()` which returns the index of the currently held value's type.
```C++

std::variant<std::string, std::vector<char>, std::vector<unsigned char>> bytes;

if(bytes.index() == 1) {
    //do something with std::vector<char>
}

if(std::holds_alternative<std::string>(bytes)){
    auto str = std::get<std::string>(bytes);
    if(!str.empty()){
        bytes.emplace(std::vector{str[0]});
        // constructs a new value in-place
    }
} 
```
As shown in the example, variants also have the `emplace()` member function for constructing a value directly in the variant.

We can use `std::visit` to perform a function templated on the type the variant holds. Using *constexpr if* we can perform a different action based on what the variant holds. Constexpr if is an if statement that is evaluated at compile time. Thus, when using constexpr if, the compiler can determine which branch is taken and inline that choice directly in the code. This makes it look like in the compiled code that there wasn't a conditional to begin with.
```C++
std::variant<char, int, double> getGpa();

auto variant = getGpa();
std::visit([](auto&& gpa) {
    using T = std::decay_t<decltype(gpa)>;
    if constexpr (std::is_same_v<T, char>) {
        printf("Letter scale\n");
    }
    else if constexpr (std::is_same_v<T, int>) {
        printf("Numeric scale\n");
    }
    else if constexpr (std::is_same_v<T, double>) {
        printf("4.0 scale\n");
    }
}, variant);
```
What `std::visit` does is call the supplied function passing in the value held by the variant. Notice the use of `auto&&`, which is essentially a universal reference because visit will pass in different types. So this lambda is essentially a template of sorts. A normal template function however wouldn't work, because you'd need to know which instantiation to call, and the fact that we don't have this information is one of the reasons we're using `std::visit` in the first place. `std::visit` essentially applies the *visitor* pattern. Here we use one function with constexpr-if to simulate having multiple overloads. We also need the `std::decay_t`, which will remove the reference if the function is passed an lvalue. We don't need it for this example but it also decays arrays into pointers. 

A variant cannot be empty. But if wanted it to be able to hold anything, we can allow it to hold `std::monostate` which is basically an empty struct designed to indicate an empty variant. 

```C++
std::variant<std::vector<int>, std::string, std::monostate> var = std::monostate{};

if (std::holds_alternative<std::monostate>(var)) {
    // empty variant
}
```

## Union

I mentioned that `variant` is a type safe union. So you may be wondering what a union is. Well basically, it's the C way of holding a value of one or multiple types all within the same area of memory. All members of a union are stored in the same memory location, therefore a union only takes up the amount of space needed for its largest member.

```C++

union MyUnion {
    int num;
    std::string name;
    unsigned flag : 1;
    unsigned long long id;
};

MyUnion un;
un.name = "Peach";
sizeof(MyUnion) == sizeof(std::string); // largest member

un.flag = 0;
```

A common usage for `union` is to set the memory as one type, and read it off as another to essentially perform a `reinterpret_cast`. This is called *type punning* and is **undefined behavior**. However, this is perfectly legal in C. Thus in C++, you should only read from a member of the same type as the actual stored value. Given this, I cannot think of a situation where a union would be preferable to `std::variant`.

# Any

`std::any` is another utility type similar to `variant`, `optional`, and `tuple`, but `any` holds only 1 value of *any* type, as the name may suggest. We can check if an `std::any` has a value via the `has_value()` member function. We can also get the `std::type__info&` of the stored type via the `type()` member function. Mutating the `any` can be done with the assignment operator, `swap()`, or `emplace()` just like a variant.

```C++
std::any any = 1;
std::cout << any.type().name() << std::endl;
//compiler dependent but likely "int"
any = 3.14;
std::cout << any.type().name() << std::endl;
//compiler dependent but likely "double"

if(any.has_value()){
    any.emplace<std::string>("something new");
}
```
Getting a value stored from an any can be done with `std::any_cast`. We can also construct an any with s`td::make_any`. `any_cast` allows type safe retrieval of the value in the any. It will throw an exception if the RTTI of the type parameter and the stored value do not match.

```C++
auto anything = std::make_any(20);

int n = std::any_cast<int>(anything);
```