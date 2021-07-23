# Policies

Taking the term from Andrei Alexandrescu, a policy is a small little class that handles a single behavior or structural element for larger, more complex classes. *Policy Based Design* is the methodology to design classes by using such policies. Policy based design is sort of like the generic programming equivalent of object composition with class hierarchies. Let's look at an OOP and generic way of abstracting the thread safety and comparator behavior of a Linked List:

```C++
// OOP object composition (untested)
class ThreadingPolicy {
public:
    virtual ~ThreadingPolicy() = default;
    virtual std::unique_ptr<LockProxy> lock() = 0;

    struct LockProxy {
        virtual ~LockProxy() = default;
    };
};

class MultithreadedPolicy : public ThreadingPolicy {
    std::mutex mu;
public:
    class MTLock : public ThreadingPolicy::LockProxy {
        std::mutex &mu;
    public:
        MTLock(std::mutex& mu) : mu(mu) {
            mu.lock();
        }

        ~MTLock() {
            mu.unlock();
        }
    };

    std::unique_ptr<LockProxy> lock() override {
        return std::make_unique<MTLock>(mu);
    }
};

class SingleThreadedPolicy : public ThreadingPolicy {
public:
    struct SSLock : public ThreadingPolicy::LockProxy {};

    std::unique_ptr<LockProxy> lock() override {
        return std::make_unique<SSLock>();
    }
};

struct Comparator {
    virtual ~Comparator() = default;
    virtual int compare(int, int) = 0;
};

struct LessThanComparison : Comparator {
    int compare(int a, int b) override {
        return a - b;
    }
};

class LinkedList {
    std::unique_ptr<ThreadingPolicy> tp;
    std::unique_ptr<Comparator> cmp;

    struct node {
        int data;
        std::unique_ptr<node> next;

        node(int data) : data(data), next(nullptr) {}
    };

    std::unique_ptr<node> root;
public:
    LinkedList(std::unique_ptr<ThreadingPolicy> && tp, std::unique_ptr<Comparator> && cmp) :
        tp(std::move(tp)), cmp(std::move(cmp)), root(nullptr) {}

    void push_back(int data) {
        const auto lk = tp->lock(); // user passed instance controls locking
        auto* n = &root;
        while(*n) {
            n = &n->next;
        }
        *n = std::make_unique<node>(data);
    }

    bool find(int e) const {
        const auto lk = tp->lock();
        auto* n = &root;
        while(*n) {
            if (cmp->compare(n->data, e) == 0) // user tells us how to compare
                return true;
            n = &n->next;
        }
        return false;
    }
};
```

There's a few problems with this approach. Firstly, we cannot have templated virtual functions. This means we'd need to manually create comparators for every type of arguments we'd want to support. Instead of an interface, we could accept a generic `std::function` to ameliorate that. Secondly, we are a bit limited. What if we wanted to have a per node locking system to improve the concurrency of the list? We could make the threading policy follow the *prototype* OOP pattern. Notice how this OOP approach however requires 2 parallel hierarchies, and because of the nature of polymorphism, indirection due to dynamic dispatch. This might be manageable, but what if we decide to make allocation something the user can control? Once again we'd be off creating another set of parallel class hierarchies.

We can make OOP work, but a better solution, especially when we're already dealing with generic code is to use policies. Customizable behaviors such as object allocation, comparators, threading, different algorithm implementations, etc. can be *policies* passed as template arguments. Each policy would adhere to a certain concept allowing the usage of many different, unrelated types. For example, a threading policy would no longer have to have a `lock()` function returning an implementation of a specific interface, but rather any object adhering to a certain concept. In our previous example, the concept would simply be `Destructible`.

```C++
#include <mutex>
#include <memory>
// Policy Implementations:
// these adhere to the concepts required of them
// kind of like OOP concrete classes
struct SingleThreadPolicy {
    char lock() const { return 0; }
    // the thread policy concept (see below) requires we return something swappable and
    // destructible
    // char satisfies this requirement and is essentially
    // a placebo for a lock
};

class MultiThreadPolicy {
    mutable std::mutex lk;
    // mutable allows lk to be mutated from const members
public:
    auto lock() const { return std::unique_lock<std::mutex>(lk); }
    // unique lock is RAII for a lock
    // locks on construction, unlocks on destruction
    // move-only like unique_ptr
    // we'll talk more about this later
};

enum class Ordering {
    less,
    equal,
    greater
};

template<typename T>
struct BasicComparator {
    static Ordering compare(const T& a, const T& b) {
        if (a < b) return Ordering::less;
        else if (a == b) return Ordering::equal;
        else return Ordering::greater;

    }
};
///

// C++17 policy definitions
// define what concepts we need our policies to have
// analogous to an OOP abstract class/interface

template<typename T>
using strip_t = std::remove_cv_t<std::remove_reference_t<T>>;

template<class Comparator, typename T, typename = void>
struct IsComparator : std::false_type {};

template<class Comparator, typename T>
struct IsComparator<Comparator, T, std::enable_if_t<
    std::is_same_v<Ordering,
    strip_t<decltype(Comparator::compare(std::declval<T>(), std::declval<T>()))>>
    // checks for a static member function compare() that operates on two Ts and returns an Ordering
    // does not have to be same const, reference, pointer, or volatility-ness
    >> : std::true_type{};

template<typename T, typename = void>
struct IsThreadingPolicy : std::false_type {};

template<typename T>
struct IsThreadingPolicy<T,
    std::enable_if_t<
    std::is_swappable_v<decltype(std::declval<const T>().lock())>
    // checks for a const member function lock() that returns a type that is swappable
    >> : std::true_type{};

template<typename T>
constexpr auto inline is_threading_v = IsThreadingPolicy<T>::value;

template<template<typename> class Cmp, typename T>
constexpr auto inline is_comparator_v = IsComparator<Cmp<T>, T>::value;

///


// primary definition of class, if it gets to this (specialization failure), class is incomplete and cannot be used
template<typename T, 
    class ThreadingPolicy = SingleThreadPolicy, 
    template<typename> class Cmp = BasicComparator, 
    typename = void>
class LinkedList;

// this template<typename> syntax indicates that Comparator is a "template-template" parameter
// a template paramater that is itself a template
// more on this later

// sfinae specialization
template<typename T,
    class ThreadingPolicy,
    template<typename> class Cmp>

class LinkedList<T,
    ThreadingPolicy, Cmp,
    std::enable_if_t<is_threading_v<ThreadingPolicy>&& is_comparator_v<Cmp, T>>>
{
    class node {
        ThreadingPolicy localLock;
        T data; ///< Invalid iff empty is true
        bool empty;
    public:
        std::unique_ptr<node> next;
        ///Invariant: next is null iff empty is true
    public:

        node() : data(), empty(true), next(nullptr) {}
        node(T&& data) : data(std::forward<T>(data)), empty(false),
            next(std::make_unique<node>()) {};

        void set(T&& data) {
            this->data = std::forward<T>(data);
            empty = false;
            if (!next) {
                next = std::make_unique<node>();
            }
        }

        inline auto lock() { return localLock.lock(); }
        inline auto isEmpty() { return empty;  }
        inline const auto& getData() { return data;  }
    };

    std::unique_ptr<node> root;
public:
    LinkedList() : root(std::make_unique<node>()) {};

    template<typename U>
    auto push_back(U&& data) -> std::enable_if_t<std::is_same_v<strip_t<U>, T>> 
    {
        // if we used T, data would not be a universal reference since
        // the function itself is not a template and T is the class template argument
        using std::swap;
        auto n = &root;
        auto lk1 = n->get()->lock();
        while (!n->get()->isEmpty()) {
            n = &n->get()->next;
            auto lk2 = n->get()->lock();
            swap(lk1, lk2);
            // hand over hand locking (we'll talk about this later)
            // lk1 and lk2 are swapped, lk1 then goes out of scope
            // releasing the lock
        }
        n->get()->set(std::forward<T>(data));
    }

    bool find(const T& e) const {
        using std::swap;
        auto n = &root;
        auto lk1 = n->get()->lock();
        while (!n->get()->isEmpty()) {
            if (Cmp<T>::compare(n->get()->getData(), e) == Ordering::equal)
                return true;
            // we don't need an instance of the comparator
            // its member function is static
            n = &n->get()->next;
            auto lk2 = n->get()->lock();
            swap(lk1, lk2);
        }

        return false;
    }
};
```

We've gained finer grain locking with policies because when we pass a type via template parameter, we have more control. We can easily construct instances of the locking policy wherever we see fit without forcing the locking policy to implement the prototype pattern. We also don't need to have an instance of the comparator. And probably most importantly, we cut down on the amount of classes we have to create. Supporting a different type is as easy as changing a single template parameter. We use SFINAE to enforce that the template parameters adhere to the policy interface. In C++20, concepts makes this much easier and cleaner.

Notice that these policies are general enough to apply to many different classes. 

Template-template parameters and more policy design examples will come later, but for now, I want to focus on using these ideas to customize the STL.

STL uses this design a lot. We've already seen it in action with Deleters on smart pointers and comparison predicates in STL containers. One thing to note is that the STL wasn't so kind as to use SFINAE to prevent incorrect concept implementations. 

A common STL policy is an allocator. These are typically the last template argument (because they are typically the least frequently customized) and are pretty similar to custom deleters with smart pointers. I won't discuss them here, but it's good to know they exist.

Let's start with an `std::map`. `std::set` is an ordered container typically implemented as a RB Tree. If we want to have a map of students, we'd need to be able to order them.

```C++
struct Student {
    std::string name, school, college, major;
    float gpa;
};

```
The concept for a comparator (`Compare` named requirement) requires a type that takes in a template parameter and defines an `operator(const T&, constT&)` which returns a bool (or something implicitly convertible to bool). It also requires that `!cmp(a, b) && !cmp(b, a)` establishes that `a == b`. The default ones provided are `std::less` and `std::greater` with use `operator<` and `operator>` respectively. Containers will order their elements so that `a` is before `b` if `cmp(a, b)` returns true. Thus, an out-of-the-box `std::set` will order elements least to greatest using the type's defined `operator<`.

Say we wanted to dehumanize our students and order them from highest to lowest gpa. Since equal gpa's should be allowed, we'll need an `std::multiset`. We could do this relatively simply by defining an `operator>` and changing the template parameter.

```C++
struct Student {
    std::string name, school, college, major;
    float gpa;

    bool operator>(const Student& other) const {
        return gpa > other.gpa;
    }
};

std::multiset<Student, std::greater<Student>> students;
```

Now let's say we wanted a multiset of students in an order that's different from the typical method to compare students: all CS students first, then alphabetically by major. we probably don't want to change `operator>` since users of our type wouldn't expect such an ordering when using the comparison operators. So instead, we'll just change the comparator on the set.

```C++

struct StudentMajorOrdering {
    // const Student& because that will bind to all references
    bool operator()(const Student& a, const Student& b) const {
        if (a.major == "Computer Science" && b.major != a.major)
            // two students that are equal must not compare to true
            return true;
        else
            return a.major < b.major;
    }
};

std::multiset<Student, StudentMajorOrdering> students;
```

The STL comparator is not a template template parameter which allows us to easily implement specific orderings like this one. However it also means that when we do use template comparators like `std::less`, we'll have to pass the element type as a parameter to both the container and comparator as well.

A difference between OOP and generic interfaces is that these two sets are not the same type. Unless our class defines generic member functions (which the STL does not), we cannot easily convert the two types and any code wishing to abstract away the specific implementation would need to use templates.

```C++

template<typename T, class Cmp, class Alloc>
void useMS(const std::multiset<T, Cmp, Alloc> &) {}

// OR

template<typename T>
void useSet(T&&) {}
// this one should probably use concepts to enforce 
// the interface T must adhere to.
```

We already saw how we can specialize `std::hash` to make custom types work with hash containers, but what if we just wanted one `std::unordered_map` with a custom hash function? We can change the hashing and the comparison templates in a similar way to changing a comparator. Let's saw we wanted a map keyed on a student where we'll have only 1 student per school. That is to say two students are equal if they go to the same school. We'll need to change the hash function and the predicate used to compare for equality.

```C++
struct StudentSchoolEquality {
    bool operator()(const Student& a, const Student& b) const {
        return a.school == b.school;
    }
    // return true iff the passed arguments are equal
};

struct StudentSchoolHash {
    size_t operator()(const Student& s) const {
        return std::hash<std::string>{}(s.school);
        // just hash the student's school member
    }
    // a hash function should return std::size_t
}

std::unordered_map<Student, std::string, StudentSchoolHash, StudentSchoolEquality> map;
```

Notice how the `operator()` member functions are `const` and are passed `const` references. Because they do not modify any outside state or have side effects, these functions are known as *pure* and should be preferred to non-pure. For those familiar with functional programming, a pure function is like a step below a functional function. They don't have side effects or modify outside state, but they can modify internal local state and have things like loops.

For the last example, let's implement a case insensitive string. We can customize `std::string` which is a type alias for `std::basic_string<char>`. The full template definition of `std::basic_string` is as follows:
```C++
 template<class charT,
           class traits = char_traits<charT>,
           class Allocator = allocator<charT> >
      class basic_string;
```
So writing a case insensitive string really just boils down to implementing a `char_traits` policy. `std::char_traits<T>` is a struct that defines quite a few <u>static</u> member functions such as `eq(), lt(), compare(), copy(), length(), move(), eof()`, and more. Unlike before where we created an entire implementation of the concept, this time we'll use inheritance to borrow the implementation of the default `std::char_traits<T>` and just shadow what we want to rewrite. Although `char_traits` is not meant to be inherited from (no virtual destructor), this is safe because we never instantiate either of them. We only use their static methods.
```C++
// Herb Sutter's GOTW #29
struct ci_char_traits : public char_traits<char> {
    static bool eq(char c1, char c2) { 
        return toupper(c1) == toupper(c2); 
    }

    static bool ne(char c1, char c2) { 
        return toupper(c1) != toupper(c2); 
    }

    static bool lt(char c1, char c2 ){ 
        return toupper(c1) < toupper(c2); 
    }

    static int compare(const char* s1, const char* s2, size_t n) {
        return memicmp(s1, s2, n);
        // case insensitive str compare
    }

    static const char* find(const char* s, int n, char a) {
        while( n-- > 0 && toupper(*s) != toupper(a) ) {
            ++s;
        }
        return s;
    }
};

using ci_string = std::basic_string<char, ci_char_traits>;
ci_string s1 = "HeLlO";
ci_string s2 = "helLO";
s1 == s2; // true
```
One thing to remember is that `std::vector<int>` is not the same type as `std::vector<char>` and thus `std::basic_string<char>` is a different type from `std::basic_string<char, ci_char_traits>`. Therefore, using things like `operator+()` or `operator<<()` will require you to convert to a c string with `c_str()` or write functions that work for the new type.

This implementation only supports `char`, but does a good job getting the point across.