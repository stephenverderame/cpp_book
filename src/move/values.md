# Alphabet Soup of Values

What we've seen so far are *rvalues* and *lvalues*. Both are these come from pre C++11 and the names originate from the idea than an lvalue can go on the left side of the equals side and rvalues may only go on the right side. A little better criteria is that an lvalue is something you can take an address of and an rvalue you cannot. Another possible distinction is that rvalues are temporary values while lvalues are named values.

```C++
int age = 5;
Person p = Person("Bob", 20);
int ageBob = p.getAge();
long long bigAge = age;
```
Here, `age, p, ageBob, and bigAge` are lvalues. `5`, the result of the `Person` constructor, and the temporary result of `p.getAge()` are all rvalues. Once again, you can think of rvalues as temporary values. We can't take the address of the result of the `Person` constructor, `5`, nor the result of `p.getAge()`. Although we can do this:

```C++
int foo();
&foo; // adress of foo
// not the address of the result of calling foo

int * resPtr = &(foo()); // error
// return from function is not an lvalue
```

Although we cannot have pointers to rvalues, we can have references to rvalues. Lvalue references are denoted with `&` and rvalue references are denoted with `&&`. Now confusingly, the rvalue reference itself is an lvalue. This lvalue just so happens to store a reference to an rvalue. Using rvalue references, we can avoid an excess construction when using rvalues.

```C++

auto makePerson() {
    return Person();
}

auto funcL(const Person & p) {

}

auto funcR(Person && p) {

}

Person p;

funcL(p); // no construction
funcR(p); // error, p is not an rvalue
funcL(makePerson()); // construct a new Person from the temporary return, pass the reference of this new Person to funcL
funcR(makePerson()); // no construction
```

As you may notice, this `&&` syntax is exactly what we use in the move constructor. As an rvalue reference indicates a temporary or value we don't care about, this means that when passed to a move constructor, we're allowing that constructor to steal the rvalues internals and leave behind a "shell" of an object.

In order to move, we need a non-const rvalue reference (otherwise we couldn't modify the old value to invalidate it and we'd effectively perform a copy). Thus, since rvalue references are primarily used for moving, we don't often use `const` rvalue references. This is because, if we tried to move construct from a `const` rvalue reference, the compiler would silently perform a copy since a `const &&` cannot bind to `&&` but it can bind to `const &` (copy constructor!).

```C++

Person getPerson() {
    Person newPerson;
    //...
    return newPerson;
}

// accepts an rvalue reference!
void doStuff(Person && p) {
    std::cout << p.getName() << "\n";
}

void doStuff2(const Person & p) {}

doStuff(getPerson());
doStuff2(getPerson()); // construct Person from return of getPerson

doStuff(Person("Joe", 10));
doStuff2(Person("Joe", 10)); // construct another Person from constructed person

doStuff(std::move(lvaluePerson));
lvaluePerson.getName(); // BAD, lvaluePerson has been moved
// internals could be empty
```

Notice `getPerson()` doesn't return a `Person&&` nor do we explicitly move `newPerson` but the result is still to move `newPerson` and bind that temporary by the rvalue reference. It's generally not a good idea to have a return type as an rvalue reference nor should we explicitly return by `std::move()` because these inhibits some compiler optimizations we'll talk about later.

So what's the point of all this? Well, since we are using a reference to an existing object, we don't have to reconstruct the object from the temporary. Furthermore, an rvalue indicates a temporary object. Thus, when we use rvalues we get to assume they the object is being discarded. Thus, we can construct new objects from rvalues very cheaply. Instead of having to copy the state from the existing object to the new instance, we can sort of just steal the state of the temporary and take ownership of it. After all, an rvalue is just a temporary about to be destroyed. 


```C++
auto sumVec(const std::vector<int> & v) {
    return std::accumulate(v.begin(), v.end());
}

// rvalue overload
auto sumVec(std::vector<int> && v){
    std::vector<int> * ptr = &v;
    // this is valid, so v is an lvalue


    return sumVec(v);
    // calls lvalue reference version
    // since v is an lvalue
}
std::vector<int> * invalid = &std::vector<int>({20, 30, 40, 560}); // error! not an lvalue

const auto sum = sumVec({20, 30, 40, 560}); // bind to rvalue overload
// the vector is constructed once, that single object is passed by reference to both functions


std::vector vec = {1, 2, 3, 4};

const auto sum2 = sumVec(vec); // calls lvalue overload
// again vector is constructed once (at its definition)
```

Later we'll see how universal references (aka forwarding references) can be used to have one function for both lvalues and rvalues. But for now, a good method (and sometimes preferable to forwarding references) to get the best of both worlds is to create two functions and implement the rvalue reference overload in terms of the lvalue one.

Now since an rvalue reference argument is an lvalue, if we want to pass it to another function taking an rvalue reference we'd have to use `std::move`.

```C++
auto func1(Car && car) {
    //...
}

auto func2(float mpg, Car && car) {
    // ..
    return func1(std::move(car));
    // cannot use car after this line
    // totally fine since that's the last line
    // in the function
}

func2(30.3, {"Tesla", "Model S"});
// A Car is constructed once, a reference to that is passed to func2
// then that same car is moved to func1
```

## More Values!

Now lvalues and rvalues aren't the full story. The taxonomy of expressions into these different types of values is known as an expression's value category. The rvalues I've been talking about so far (and pre C++11 rvalues) are really *prvalues* and the new rvalue category contains both *prvalues* and *xvalues*. The lvalues I've been talking about (pre C++11 lvalues) are still called lvalues but are now a subset of expressions categorized as *glvalues*. (I did name this chapter "alphabet soup" for a reason).

```
             expression
            /          \
        glvalue       rvalue
       /      \      /      \
 lvalue        xvalue        prvalue
```

### prvalues

A *prvalue* is the result of the construction of an object including functions returning non-references, literals (except string literals), enumerators, lambda expressions, pending member function calls, and the `this` pointer. Roughly speaking it is a direct value. Therefore, a prvalue cannot be polymorphic so its static type must be the same as its dynamic type.

```C++
Person("Jimmy", "Longbottom"); // prvalue
{20, 30, 40}; // consruct an std::initializer_list<int>
std::string("Hello World"); // prvalue

"Hiya"; //not prvalue

str1 + str2; // operator+ constructs a new string, that result is a prvalue

42 // literals are prvalues

struct Person {
    enum {
        brown,
        black,
        blonde,
        red,
        ginger
    } hairColor;

    auto guessEyeColorFromHairColor() {
        switch (hairColor) {
            // ...
        }
    }
}

Person::ginger; // prvalue, since enumerators are prvalues

Person p;

p.guessEyeColorFromHairColor; // pending member function call
// prvalue
// cannot do anything with this except call it
// cannot take reference of this

// as a reminder, here is the member function pointer syntax
HairColor(Person::* memberPtr)(void) = &Person::guessEyeColorFromHairColor;
(p.*memberPtr)(); // call it with p as the context object

// But
p.*memberPtr; // is also a prvalue (same reason as above)
```

### xvalues

Named as such because it is an "expiring" value. It denotes an object or bit-field is nearing the end of its lifetime and can be moved. These were originally classified as lvalues pre C++1, so one way to think of them is lvalues (or technically glvalues now) that can be moved. They are created in 3 cases: accessing a non-static member of an rvalue, the result of a function call that returns an rvalue reference, or casting an object to an rvalue.

rvalues are composed of prvalues and xvalues.

```C++
auto getPerson() {
    return Person {Person::red};
}

Person&& bar() {
    // ...
}

getPerson().hairColor; //xvalue

Person p {Person::brown};

std::move(p); // we'll discuss this later but std::move actually just casts to an rvalue
// so this is an xvalue
static_cast<Person&&>(p); // also xvalue

bar(); // xvalue (result of calling a function returning an rvalue reference)


```

### glvalues

Glvalues are "generalized lvalues" and are composed of lvalues and xvalues. They can be polymorphic (actual type is not the static type) and have an incomplete type. When a glvalue appears where an prvalue is expected, the glvalue is converted to a prvalue.

Let's look at an example to recap:

```C++
class Game {
    std::string name;
public:

    Game(std::string some_name) : name(std::move(some_name) /*xvalue*/) {}

    std::string& getName()
    {
        return name;
    }

    std::string cpyName() const
    {
        return name;
    }
};

Game g("Skyrim"); 
g; // lvalue

g.getName(); // lvalue
g.cpyName(); //prvalue
```

### Returning by rvalue reference

Now I said it's *generally* not a good idea to return by rvalue references, yet I also said that the result of a function that returns by rvalue reference is an xvalue. So clearly, returning by rvalue references has its places. And yes, yes it does but its usage is somewhat esoteric. 

Take the following code snippet:

```C++
class MaybeResult {
    ResultType result;
    bool success;
public:
    MaybeResult() : success(false) {}
    MaybeResult(const ResultType & res) : result(res), success(true) {}

    ? valueOrThrow() {
        // just for fun: this concept is called a Monad and this particular function is often named unwrap()
        if (success) return result;
        throw std::runtime_error("No value stored in result");
    }
};

MaybeResult doComputation() {
    // ...
    return MaybeResult(computation);
}

auto res = doComputation().valueOrThrow();
```

Now, how can we avoid copying `result`. Perhaps `ResultType` is an absolute unit of a matrix, it would be great not to copy it. But if we return by reference, then the reference will dangle since `result` is a member of a temporary. What if we could move `result`? Well, that would be lovely but then `valueOrThrow` would be unsafe to call on an lvalue!

Thankfully, we can use *ref-qualifiers* to constrain the type of value the member function is applied on. Ref-qualifiers work exactly like declaring a function `const`. In truth, both qualifiers aren't properties of the function but rather constraints on the type of the implicit first argument. For example, a free function version of `valueOrThrow` could be:

```C++
ResultType&& valueOrThrow(MaybeResult && maybe) {
    if (maybe.success) return std::move(maybe.result);
    else //...
}

ResultType& valueOrThrow(MaybeResult & maybe) {
    if (maybe.success) return maybe.result;
    else //...
}

const ResultType& valueOrThrow(const MaybeResult & maybe) {
    if (maybe.success) return maybe.result;
    else //...
}
```

The analogous member functions using ref-qualifiers would therefore be:

```C++
    //...


    // called on rvalues
    ResultType&& valueOrThrow() && {
        if (success) return std::move(result);
        else //...
    }

    // when valueOrThrow() is called on lvalues
    ResultType& valueOrThrow() & {
        if (success) result;
        else //...
    }

    // when called on const lvalues
    const ResultType& valueOrThrow() const& {
        if (success) return result;
        else //...
    }
};
auto res = doComputation().valueOrThrow(); // #1
MaybeResult res2;

res2.valueOrThrow(); // #2

const auto res3 = doComputation();

res3.valueOrThrow(); // #3
```





