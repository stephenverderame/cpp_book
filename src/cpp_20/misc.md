## Rocketship Operator

This is actually called the 3-way-comparison operator, but it's colloquially known as the Rocketship Operator.
This is basically a type-safe version of Java's `comareTo()` method.

If you've ever implemented a comparable type, you may know the pain of manually defining all `<=, ==, !=, >= <, >` and then defining them again so that they are commutative.
If so, then the three-way comparison `( <=> )` is what you need! Given the following syntax:
`a <=> b`
It returns an object that is implicitly convertible to an `int` that is:
* `< 0` if `a < b`
* ` = 0` if `a == b`
* ` > 0` if `a > b`
What it actually returns is an object of type `std::strong_ordering` or `std::partial_ordering`, which are convertible to integers.
`std::strong_ordering` has 3 members, `std::strong_ordering::equal`, `less`, and `greater`.
`std::partial_ordering` has 4: `equivalent`, `less`, `greater` and `unordered`.

The nice thing about our little rocketship here, is that we don't have to explicitly use it to take advantage of it. Consider:

```C++
#include <compare>
struct Person {
    std::string name;
    int age;
    
    Person(std::string n, int a) 
        : name(n), age(a) {}
    
    auto operator<=>(const Person& other) {
        return age <=> other.age;
    }
    
    auto operator<=>(int age) {
        return this->age <=> age;
    }

    auto operator<=>(char c) {
        if (name.empty())
            return std::partial_ordering::unordered;
        c = toupper(c);
        if (auto first_letter = toupper(name[0]); first_letter < c)
            return std::partial_ordering::less;
        else if (first_letter == c)
            return std::partial_ordering::equivalent;
        else
            return std::partial_ordering::greater;
    }
}

Person p {"PersonA", 20};
Person p2 {"PersonB", 18};
Person p3 {"ABB", 18};

if(p2 < p) {
    //this executes
}
if(100 >= p) {
    //this executes
}
auto val = p2 == p3; //true
```
The compiler uses `operator<=>` to generate definitions for all other comparison functions! It also uses this to generate communative versions of each comparison operator as well!
Traditionally, if we defined `>=` for an `int` and our type,
we'd have to define two versions.
One where the integer is the left-hand parameter, and the other where the integer is the right.
With the spaceship operator, the compiler handles that for us.
Indeed, what's actually going on is the compiler is generating all the comparison functions for us. 

There is a slight problem. For some types, such as strings, some comparison operations are much faster than others, namely `==` and `!=`.
If two strings aren't the same size, they cannot be equal, and we can do this check quickly before lexicographically comparing them character by character.
However, determining if one string is less than another always requires an `O(n)` lexicographic comparison.
The good news is that's perfectly fine because the compiler will implement `==` independently of the spaceship operator and handle that for you.
If you had your own type with your own equality comparison that wasn't simply delegating the comparison to a member, you can simply define your own operator==().
Any manually defined comparison operators will take precedence over the rocketship operator.