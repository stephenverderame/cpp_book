# Templates

We've been using these quite a bit already, and you may already be familiar with things like templates from another programming language.
Templates allow us to pass parameters (often types) to classes, structs, and functions at compile time. 
Effectively, what this does is replace every template argument with whatever type is passed to it. 
For example, `std::vector` is a template that is *instantiated* with the type we specify.
This allows one definition of `std::vector` to be used for many types.
Each template *instantiation* is a different type; behind the scene each unique combination of template parameters passed to a template type results in 
the compiler generating a unique class/function. 
As we've seen, this means that `std::unique_ptr<Base>` and `std::unique_ptr<Derived>` are totally distinct classes even though `Derived` is a subclass of `Base`.

```C++
template<typename T> // this is the template parameter
class MyVec {
    size_t size;
    T * data;
public:
    explicit MyVec(size_t size) : size(size), data(new T[size]) {}
    // T is effectively replaced with whatever type we specify
    // this type substitution happens at compile time allowing
    // operations like new (which need the static type)
    // to work
    ~MyVec() {
        delete[] data;
    }

    T& operator[](size_t idx) {
        return data[idx];
    }

    T* begin() {
        return data;
    }

    T* end() {
        return data + size;
    }

    bool contains(const T& e) const {
        const auto end = data + size;
        return std::find(data, end, e) != end;
    }
};

MyVec<int> mv(100);

// -----------------------------------------------
// Effectively what happens is the compiler generates the following:

class MyVec_int {
    size_t size;
    int * data;
public:
    MyVec_int(size_t size) : size(size), data(new int[size])
    ~MyVec_int() {
        delete[] data;
    }

    int& operator[](size_t idx) {
        return data[idx];
    }

    int* begin() {
        return data;
    }

    int* end() {
        return data + size;
    }

    bool contains(const int& e) const {
        const auto end = data + size;
        return std::find(data, end, e) != end;
    }
};

MyVec_int mv(100);
```

In order to instantiate a template, the template's full definition (not just declaration) must be available. 
Since translation units (source file + all included headers) are compiled independently,
the general result is that a template function or class must be defined in a header file. 
This allows multiple source files to declare instances of template types because when the compiler compiles the translation unit,
the definition is available since it's in the header file. 
There is a way around this however.
If the amount of different instantiations of a template is limited,
you can define the class in a source file and manually enumerate through all allowed instantiations in that file. 
This way, when another translation unit uses an instance of a template class,
the linker can link the instance defined in an external translation unit with its usage.

```C++
// my_vec.h
template<typename T> 
class MyVec {
    size_t size;
    T * data;
public:
    explicit MyVec(size_t size);
    ~MyVec();
};
```

```C++
// my_vec.cpp
#include "my_vec.h"

template<typename T>
MyVec<T>::MyVec(size_t size) : size(size), data(new T[size]) {}

MyVec<T>::~MyVec() { delete[] data; }

// Manual instantiations
// full definition avaliable here
template MyVec<int>;

template MyVec<double>;
```

```C++
// main.cpp
#include "my_vec.h"

// only declaration is available
MyVec<long long> v(100); // ERROR
// no definition available

MyVec<int> mvi(10);
// fine, link with the instantiated template from another
// translational unit

```

As we've seen, template arguments can be omitted when the compiler can infer them.

```C++

std::vector vec = {1, 2, 3, 4};
// infers std::vector<int>

template<class T> // the keyword class can also be used, there is no difference at all although class conveys more that you are expecting a class
T add(T a, T b) {
    return a + b;
}

const auto sum = add(100, 200);
// deduces to add<int>(100, 200)
```

Notice that `add` may not work with every type we pass to it. For example if we passed two vectors, we'd get a compiler error. 
However, even with this failure case, `add` still compiles. 
A template instantiation is not generated unless that instantiation is *ODR-used* (used in a place where a definition is required, we'll discuss this later). 
Later we'll also see ways to constrain the template arguments because if you did ODR-use an invalid template instantiation you'd likely be greeted with a somewhat cryptic and perhaps very lengthy error message.

Because we use `T` for the return types and both arguments, all 3 must be exactly the same type. If that wasn't the case, we'd need more type parameters.

```C++
template<typename T, typename U, typename ReturnType>
ReturnType add(const T& a, const U& b) {
    return a + b;
}

std::string hello = "Hello";
auto result = add(hello, 5);
// We'll discuss type deduction later, but this deduces to
// add<std::string, int, std::string>
```

We can also have non-type template parameters too. We saw this with `std::array`.

```C++
template<typename T, int N>
std::ostream& operator<<(std::ostream & str, const T (& arr)[N]) {
    str << "[";
    for (auto i = 0; i < N - 1; ++i) {
        str << arr[i] << ", ";
    }
    if (N >= 1) {
        str << arr[N - 1];
    }
    str << "]";
    return str;

}
// pass by reference to T array of size N

int myArray[] = {1, 2, 3};
std::cout << myArray;
// "[1, 2, 3]"
```

Let's look at another example.

```C++
#include <iostream>
#include <memory>
#include <optional>

template<typename T>
class LList {
    struct node {
        // fully qualified name is LList<T>::node
        std::shared_ptr<node> next;
        T data;

        node(const T& data) : next(nullptr), data(data) {}
        // T must be copy constructable

        node(const node& other) {
            if (other.next) {
                next = std::make_shared<node>(other);
            }
            else next = nullptr;
            data = other.data;
            // T must be copy assignable
        }
        node() = default;
        // T must be default constructable
    };

    std::shared_ptr<node> first; ///< Invariant: nullptr iff list is empty
    std::weak_ptr<node> last; ///< Invariant: nullptr iff list is empty
public:
    LList() = default;

    LList(const LList<T>& other) : LList() {
        // LList<T> is a type, LList is not
        // we make this copy constructable from lists ONLY of the same type
        *this = other;
    }

    // Deep copy, strong guarantee
    // The default copy would increment the reference count of the pointer (shallow copy)
    // So we'd end up with two linked lists sharing the same data
    LList& operator=(const LList<T>& other) {
        if (other.first) {
            first = std::make_shared<node>(*other.first);
            set_last();
        }
        else {
            first = nullptr;
            last = std::weak_ptr<node>(first);
        }
        return *this;
    }

    LList(LList<T>&&) noexcept = default;
    LList& operator=(LList<T>&&) noexcept = default;

    // Strong
    void push_back(const T& elem) {
        if (!set_first_if_empty(elem)) {
            if (last.expired())
                throw std::runtime_error("Invariant violated");
            auto lastNode = last.lock();
            lastNode->next = std::make_shared<node>(elem);
            last = std::weak_ptr<node>(lastNode->next); //noexcept
        }
    }
    // Strong guarantee
    void push_front(const T& elem) {
        const auto oldRoot = first;
        first = std::make_shared<node>(elem);
        first->next = oldRoot; //copying a smart pointer is noexcept
    }

    bool empty() const noexcept {
        return first == nullptr;
        // could have have static_cast first to bool
    }

    std::optional<T> pop_front() noexcept {
        if (first) {
            std::optional result = std::move(first->data);
            // T must be move constructable
            // safe to move because the reference to data is about
            // to be destroyed
            first = first->next;
            return result;
        }
        return {};
    }
private:
    /**
    * Creates the first node of the list if the list is empty
    * Strong guarantee
    * @param elem the data for the new first node if created
    * @returns true if elem was set to the new first node, else false
    */
    bool set_first_if_empty(const T& elem) {
        if (!first) {
            first = std::make_shared<node>(elem);
            last = std::weak_ptr<node>(first); // noexcept
            return true;
        }
        return false;
    }

    /**
    * Updates last to reference the last element in the list
    */
    void set_last() noexcept {
        auto n = first;
        auto lastNode = n;
        while (n) {
            lastNode = n;
            n = n->next;
        }
        last = std::weak_ptr<node>(lastNode);
    }
};

int main() {
    LList<int> stack;
    stack.push_front(10);
    stack.push_front(20);
    std::cout << stack.pop_front().value_or(-1) << "\n"; //20
    // value_or uses ref qualifiers to move out the contained value of the optional
    // since we call it on an rvalue
    std::cout << stack.pop_front().value_or(-1) << "\n"; //10
    std::cout << stack.pop_front().value_or(-1) << "\n"; //-1
    auto queue = stack;
    queue.push_back(100);
    queue.push_back(200);
    std::cout << queue.pop_front().value_or(-1) << "\n"; //100
    std::cout << stack.empty() << "\n"; //1 (true)
    std::cout << queue.empty() << "\n"; //0 (false)
    auto q2 = queue;
    q2.push_back(500);
    std::cout << q2.pop_front().value_or(-1) << "\n"; //200
    std::cout << q2.pop_front().value_or(-1) << "\n"; //500
    std::cout << queue.pop_front().value_or(-1) << "\n"; //200
}
```

Notice we make a few subtle assumptions about `T`. We assume that whatever type is instantiated is copy constructable, copy assignable, move constructable, and default constructable. 
One type that won't satisfy this is a unique pointer for example. What we've done is we assumed a static interface that all elements of our list must adhere to.

## One Definition Rule

Firstly, *definitions* are *declarations* that fully define something except for the following cases (and a few more):
* Function declaration without a body
* Declaration with `extern` that lacks an initial value
    ```C++
    extern const int i; // declaration
    extern const int j = 5; // definition
    ```
* Non-inline static members in a class
    ```C++
    struct S {
        int i; // definition
        static int j; // declaration
        inline static int k; // definition
    };

    int S::j; // definition
    ```
* Declaration of class without body
    ```C++
    class S;
    class ReturnType func(class ArgType a);
    ```
* Using and typedef aliases
* Template parameters
    ```C++
    template<typename T> //T is declared
    ```

Now as the name implies, only one definition of something is allowed in a single translation unit,
and one and only one definition of non-inline functions and variables must exist in a program. 
Finally, one definition of inline functions and variables must be present in every translation unit where they are ODR-used.

Functions are ODR used when somebody makes a call to it or takes its address. Objects are ODR used when its value is read (unless it's a compile-time constant), written, has its address taken, 
or a reference is bound to it. For a reference, it is ODR used when it is not known at compile time. Anything that's ODR used must have a valid definition somewhere in the program.

This was a pretty round about way of basically saying that templates are normally defined in header files and nothing else can be defined in a header file unless it's `inline` or a non-static member variable.

## Type Aliases

We can add an alternative name to types using a type alias. The old way of doing this was through the `typedef` keyword, but now we can do this with `using` declarations. 
`using` declarations may be templates themselves, and they respect access modifiers when declared as part of a class. As I just talked about, type aliases are not definitions.

```C++
template<typename T>
using stack_t = LList<T>;

stack_t<int> stack;

class Test {
    using name_t = std::string;

    name_t name;
public:
    using test_t = int;
    name_t getName() const { return stack; }
};

Test::name_t nm; // error, not visible
Test::test_t tt; // good

template<typename T>
using tuple_3 = std::tuple<T, T, T>;

tuple_3<int> tp;
// instead of std::tuple<int, int, int>
```

Like defining `const` or `constexpr` variables instead of magic values, type aliases provide an opportunity to add documentation via a name for a type and make it easy to replace types. 
For example, let's say that you want to change the stack implementation from using our `LList<T>` to an `std::deque<T>`. 
Instead of having to change every reference to the type (return type, parameter type, variable declarations, etc.) you can just change it in one place and be done with it.


### Possible Exercises

1. Make the `Ringbuffer` class from earlier into a template. Instead of using raw pointers, can you use a `std::unique_ptr`? (If you were actually implementing this, the best choice would be `std::vector`). Make sure it's copyable and moveable. Alternatively, make your favorite container with templates.

2. Make the `Vec3d` class from before into a template which is templated on type to store and number of elements. Define all of the same members as before plus make it copy constructable and assignable from `std::array`, `std::vector`, `std::initializer_list` and really all other list/vector like containers with a single constructor and assignment operator. For a bonus, do the same with a reference to an array (will need a separate overload).

3. What would happen if you casted the integer 300 to a `char`. What about the float `12.4f` to a `long long`. How would the result of the cast compare to the original value? What about if you casted the integer `500` to a `short`? How do these scenarios differ? From these ideas, see if you can implement `narrow_cast<T, U>` which casts the arithmetic type `T` to the arithmetic type `U` if the value of the `T` object can fit in `U`. This function is part of the GSL (guidelines support library) so an implementation can be found online.