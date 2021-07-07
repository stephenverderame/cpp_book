# Namespaces

Namespaces are a way to organize related pieces of code and insulate their names from 3rd parties and outside code. You can think of them as analogous to the zip code in a mailing address. There are multiple 330 Pine Tree Roads but there is only one 330 Pine Tree Road 14850. The same idea goes for namespaces.

```C++
namespace a {
    int add(int a, int b);
}

namespace b {
    int add(int a, int b);
}

// Both adds are in separate namespaces so this is ok

a::add(5, 4);
b::add(7, 3);

```

The `::` operator is used to resolve names in namespaces the same way the `.` operator resolves names in class scopes.

Sometimes it can get annoying to constantly use `x::` where `x` is a namespace name. Therefore, we can use a `using` declaration to bring the names of a namespace into the current scope and allow them to be used without qualifying them with their namespace name. This however opens up the door to name collisions once again. An alternative is to use the `using` declaration to bring into scope only certain names instead of everything in the entire namespace.

```C++
using namespace a; //bring everything from namespace a into scope
using std::cout; // bring cout into scope

cout << add(5, 4) << "\n";\
// calls a::add
// no need to qualify cout with std::

```

Now what to put in a namespace? Well, at the very least, if we have a class `X` in namespace `Y`, we should ensure that the entire interface of `X` is also in that namespace. What constitutes part of a class's interface? Well Herb Sutter's *Interface Principle* says that anything that mentions `X` or is supplied with an `X` is logically part of the interface of `X`. Thus, these things should be in the same namespace as `X`. This plays nicely with a C++ mechanic known as *ADL* (Argument Dependent Lookup) aka Koenig Lookup: during name resolution, if a function `f` is supplied with a type `Y::X`, then it can resolve the name `f` by searching in namespace `Y` and looking for `Y::f`. Therefore, any related function of a class should be in the same namespace of that class.

```C++
namespace mail {
    struct MailingAddress {
        using std::string;

        string state, street, city;
        int streetNumber;
    }

    void sendMessage(const & MailingAddress dst, const & std::string message);
    // the using declaration for std::string is not in scope here so std::string must
    // be fully qualified
}

const auto addr = mail::MailingAddress { /* ... */};
sendMessage(addr, "Hello There");
// same as 
mail::sendMessage(addr, "Hello There");
// By rules of ADL lookup, the function sendMessage does not have to be fully qualified
```