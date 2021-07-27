# Gmock

GMock provides the ability to mock implementations. For example let's say that you are testing some logic which performs some operation on data received fromm a database. To keep tests fast, and the logic verifiable, it's probably useful to have a mock database which you can control. What we can do is create a database *gateway*: an interface that abstracts away the concrete database implementation. The we can have our concrete production class and a second concrete mock class acts like a real database. Gmock makes developing such a class much easier by providing macros to generate method implementations and assert passed parameters and calling conditions (ex. how many times the function is called, when it's called, etc.).

The googletest github repo contains both gtest and gmock. So the only thing we need to do to use gmock is add gmock to the `target_link_libraries` call for our test in CMake.
```
target_link_libraries(Test PRIVATE GTest::gtest_main PRIVATE GTest::gmock)
```

Then in our test file we can include `<gmock/gmock.h>`.

We can mock methods of a class (we cannot mock free functions) with the `MOCK_METHOD` macro which takes the method return type, name, arguments surrounded by parenthesis, and also optionally and qualifiers surrounded by parenthesis such as `const`, `noexcept`, `override`, or ref qualifiers. Mock methods must always be public.

```C++
class MyMock {
public:
    MOCK_METHOD(std::string, getName, (), (const));
    // std::string getName() const;

    MOCK_METHOD(int, mult, (int, int), (const));
    // int mult(int, int) const;

    MOCK_METHOD(void, setAge, (int));
    // void setAge(int);
};
```

In a test, we can use the `EXPECT_CALL` macro to assert that a method gets called, use *matchers* to assert conditions about the passed arguments, and use *actions* and *cardinalities* to control the behavior of the mocked method. The first argument to `EXPECT_CALL` is an instance of a mock class, the second argument is the name of the method that should be called and matchers surrounded by parenthesis. The nth matcher in the parenthesis applies to the nth argument of the method call.

We discussed matchers in the last chapter, but one matcher we haven't discussed is `testing::_` which is a matcher that matches anything.

```C++
using namespace testing;
TEST(MockTest, sqrtCalled) {
    MyMock mock;
    EXPECT_CALL(mock, mult(_, AnyOf(Eq(10), Gt(100))));
    // _ indicates the first argument can be anything
    

    mock.mult(2, 300); // passes because the second argument is > 100
}
```

## Actions

This is great to check that a function is called, but let's make the mock method do something. We can specify an action to `WillOnce` or `WillRepeatedly` by chaining these function calls after the `EXPECT_CALL` macro.

```C++
EXPECT_CALL(mock, mult(_, AnyOf(Eq(10), Gt(100))))
    .WillOnce(/* action 1*/)
    .WillOnce(/* action 2*/)
    .WillOnce(/* action 3*/)
    .WillRepeatedly(/* action 4 */);
// let n be the number of times mock.mult is called
// if n <= 3, do action_n
// else do action_4
```

* `Return([optional value])` - returns from a void function or returns the passed value
* `ReturnArg<N>()` - returns the nth argument to the function (0 based)
* `ReturnRoundRobin(vectorOrInitializerList)` - circles through elements in the container, returning each one and moving to the next
* `ReturnRef(variable)` - returns reference to `variable`
* `Throw(exn)` - throws `exn`
* `Assign(&variable, value)` - assigns `value` to `variable`. Expects a pointer as the first argument
* `SaveArg<N>(pointer)` - saves the nth argument to the value pointed to by `pointer`.
* `DeleteArg<N>()` - deletes the nth argument, which should be a pointer
* `SetArgReferee<N>(value)` - assigns the nth argument to `value`. The arg should be a reference
* `Invoke(f)` - calls the callable object `f` with the parameters passed to the function
    * `Invoke(objectPointer, member function pointer)`
* `InvokeWithoutArgs(f)` - invokes a function taking no arguments
* `InvokeArgument<N>(args...)` - invokes the nth argument as a callable object passing in `args...`

We can also compose actions as well:

* `DoAll(actions...)` - self explanatory
* `IgnoreResult(action)` - ignore the result of an action
* `WithArg<N>(action)` - executions `action` with the nth argument passed
    * `WithArgs<N0, N1, N...>(action)`

More information is available [here](https://google.github.io/googletest/reference/actions.html).

So let's mock our class:

```C++
TEST(MockTest, testActions) {
    MyMock mock;
    EXPECT_CALL(mock, mult(_, AnyOf(Eq(10), Gt(100))))
        .WillRepeatedly(Invoke([](auto a, auto b) { return a * b; }));

    const std::initializer_list<std::string> names = { "Molly", "Andrew", "Jimmy", "Julia", "Kathy", "Roger" };
    EXPECT_CALL(mock, getName())
        .WillRepeatedly(ReturnRoundRobin<std::string>(names));

    int prevAge = 0;
    EXPECT_CALL(mock, setAge(Gt<int&>(prevAge))) // specify that prevAge is passed by reference
        .WillRepeatedly(SaveArg<0>(&prevAge));


    ASSERT_EQ(mock.mult(2, 300), 2 * 300); 
    mock.setAge(1);
    mock.setAge(2);
    mock.setAge(4);
    mock.setAge(10);
    //  mock.setAge(8); // error, 8 is < prevAge
    ASSERT_THAT(std::vector<std::string> { mock.getName() }, IsSubsetOf(names));
}
```

## Cardinalities

