# Function Pointers

Function pointers allow us to pass functions as values. We'll discuss this a lot more later but I did want to introduce the topic here.

Functions are static data. So **do not** free them. 
All function pointers essentially represent non owning references. Function pointers behave like normal pointers, but you can also apply `operator()` to call them like a normal function.

The syntax for a function pointer is as follows: `<return type>(*[optional name])(<comma separated list of argument types>)`.

```C++
int add(int a, int b) {
    return a + b;
}

int sub(int a, int b) {
    return a - b;
}

auto elementWiseOp(int(* func)(int, int), const std::vector<int> & a, const std::vector<int> & b) {
    std::vector<int> result;
    if (a.size() != b.size())
        return std::invalid_argument("Element wise operations must be performed on equal sized vectors");
    result.reserve(a.size()); //allocate memory but don't initialize it
    std::transform(a.begin(), a.end(), b.begin(), std::back_inserter(result), func);
    // iterates over a and b together calling `func` with an element from each
    // taking the result of `func` and inserting it into the back of `result`
    return result;
}

int(* addPtr)(int, int) = &add;
addPtr(5, 3); // add

using arit_func_t = int(*)(int, int); // type alias

arith_func_t addPtr2 = &add;

std::vector vecA = {10, 20, 30, 40, 50};
std::vector vecB = {1, 2, 3, 4, 5};

auto aSubB = elementWiseOp(&sub, vecA, vecB);
auto bSubA = elementWiseOp(&sub, vecB, vecA);
auto aPlusB = elementWiseOp(addPtr2, vecA, vecB);
```

Function pointers aren't as powerful as the modern `std::function`, which can use ANY callable object. I showed a bunch of things I haven't discussed yet, so we'll talk about that later.

Finally, function pointers to methods are a little different. 
We need to encode the method's owning class in the type of the pointer and pass an instance of that class to the function pointer via dot syntax when calling it.

```C++
class Foo {
    int doA(bool sw, int flag) {
        if (sw) return flag;
        else return flag * 100;
    }

    static int doB(int a, int b) {
        return a * b - a;
    }
}

int(Foo::* fooMethodPtr)(bool, int) = &Foo::doA;
// need the Foo:: to encode that the function pointer points to a method
// getting the address also needs the Foo:: since an unqualified doA would be a free function


Foo f;

(f.(*fooMethodPtr))(true, 10); // 10
// we must dereference fooMethodPtr and get the instance's version of the method
// we need parenthesis around the f.(*fooMethodPtr) as well so we apply the instance method and not
// the function pointer



// static method function pointers are basically the same as free function pointers

int(* myFunc)(int, int) = &Foo::doB;

myFunc(1, 2); // 1
```

Again, prefer `std::function` to dealing with function pointers directly.

# Raw Pointers

Prefer smart pointers, `std::vector`s, `std::array`s, and other modern C++ features. 
With raw pointers, you must ensure to match the corresponding de-allocation function with the allocation function used to allocate the memory.


```C++

struct Bar {
    int a, b, c;

    Bar(int a, int b, int c) : a(a), b(b), c(c) {}
}

Bar * bar = new Bar(20, 30, 40);
// arguments after new are arguments to constructor

bar->a;
delete bar;


Bar ** bars = new Bar[30];
// amount of elements passed in to new[]
// allocates an array of pointers

for(int i = 0; i < 30; ++i) {
    bars[i] = new Bar(i, 2 * i, 3 * i);
    // allocate elements
}

// later

for (int i = 0; i < 30; ++i) {
    delete bars[i]; //delete elements
}

delete[] bars; // delete array



Bar bc(10, 10, 10);
Bar * bar2 = &bc;
// ref, do not delete
```