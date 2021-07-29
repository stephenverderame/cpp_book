# Google Test

GTest is Google's C++ unit testing framework. It's not the only one out there, but it is a good one.

We'll use CMake to automatically download, generate, and build GTest from github. In the next unit. We'll discuss CMake in more detail so these may not make the most sense right now.
In the top directory, we'll want to have a `CMakeLists.txt` and a `GTestCMakeLists.txt.in` that looks like this:

CMakeLists.txt
```Python
cmake_minimum_required (VERSION 3.8)

project ("Tests")

macro(external_add filename pkname)
	configure_file(${filename} "${CMAKE_SOURCE_DIR}/external/${pkname}-download/CMakeLists.txt") 
    # Copy file specified as first argument of macro into external/gtest-download/CMakeLists.txt
    # and execute it (will download lib)

	execute_process(COMMAND "${CMAKE_COMMAND}" -G "${CMAKE_GENERATOR}" .
		WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/external/${pkname}-download")
    # Run CMake generator in download directory

	execute_process(COMMAND "${CMAKE_COMMAND}" --build .
		WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}/external/${pkname}-download")
    # Run build command in generated src directory from the previous command

	set(${pkname}_SUBDIRS "${CMAKE_SOURCE_DIR}/external/${pkname}-src"
	"${CMAKE_SOURCE_DIR}/external/${pkname}-build")
    # Set a variable called gtest_SUBDIRS
endmacro() 

#Install googletest
external_add(GTestCMakeLists.txt.in gtest)
# calls the macro with the first argument as GTestCMakeLists.txt.in
# and the second argument as gtest

set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)

add_subdirectory(${gtest_SUBDIRS})
# Run CMake in subdirectories (GTest has its own CMakeLists.txt)


enable_testing()
include (CTest)

# Enable CMake testing
# Allows CMake commands to find and run tests

# Include sub-projects.
add_subdirectory ("Tests")
```

GTestCMakeLists.txt.in
```CMake
cmake_minimum_required(VERSION 3.8)
project(gtest-download NONE)

include(ExternalProject)
ExternalProject_Add(googletest
	GIT_REPOSITORY https://github.com/google/googletest.git
	GIT_TAG master
	SOURCE_DIR "${CMAKE_SOURCE_DIR}/external/gtest-src"
	BINARY_DIR "${CMAKE_SOURCE_DIR}/external/gtest-build"
	CONFIGURE_COMMAND ""
	BUILD_COMMAND ""
	INSTALL_COMMAND ""
	TEST_COMMAND ""
	)
# Download GTest from github
```

You project directory should look something like this:

```
Tests
|   CMakeLists.txt
|   GTestCMakeLists.txt
|
|---external
|   |   gtest-download
|   |   gtest-build
|   |   gtest-src
|   |
|---Tests
|   |   CMakeLists.txt
|   |   main.cpp
|   |
|---out
```

Next, in the CMakeLists.txt file of the project subdirectory. Let's add an executable and link it with gtest

```Python
cmake_minimum_required (VERSION 3.8)

# Add source to this project's executable.
add_executable (Test1 "main.cpp")
target_link_libraries (Test1 PRIVATE GTest::gtest PRIVATE GTest::gtest_main)
# Linking with gtest_main allows us to use GTests' main function for us
add_test(Test1 Test1)
# create a test called Test1 which runs the target Test1
```

Finally, in main.cpp Let's write a basic test:

```C++
#include <gtest/gtest.h>

TEST(SuiteName, testName) {
    ASSERT_TRUE(true);
}
```

Linking with `gtest_main` allows us to not write a main method. Which, sometimes we may need but would look like this:

```C++
int main(int argc, char ** argv) {
    testing::InitGoogleTest(&argc, argv);
    return RUN_ALL_TESTS();
}
```

On Linux, you can execute tests with the command `make test`, or execute `ctest` in the build directory of the project.

In GTest, we can create a new test with the `TEST()` macro which takes as its first parameter the name of the suite and as a second parameter the name of the test. Both names must be valid C++ identifiers however they also must not contain underscores. Although we declare no variables `SuiteName` and `testName`, this still works because the macro stringifies whatever you type. Thus, the literal text entered for the suite name and test name becomes the actual name. So these arguments are not strings. Internally, this macro creates a class that uses `SuiteName` and `testName` for its name. Prepending either the suite or test name with "DISABLED_" will disable the test. Within a test, we can use gtest assertions or expectations to ensure behavior is as we expect. If an assertion fails, the test fails and exits immediately. If an expectation fails, the test fails but continues to run. Both assertions and expectations are named the same except expectations use `EXPECT_` while assertions are named `ASSERT_`.

It's generally good practice to minimize the amount of assertions per test (some developes advocate for one assertion per test). This makes it very easy to find what failed when you see that a test failed.

Let's look at some more useful assertions.

```C++
TEST(SuiteName, testName2) {
    ASSERT_EQ(100 * 3, 300); // ==
    ASSERT_NE(100, 0); // !=
    ASSERT_LE(100, 100); // <=
    ASSERT_LT(-20, 1); // <
    ASSERT_GE(20, 19); // >=
    ASSERT_GT(20, 0); // >
    
    ASSERT_FLOAT_EQ(2.5f, 5.f / 2.f); // == but takes into account float precision loss
    ASSERT_DOUBLE_EQ(2.5, 5. / 2.); // same as above but for doubles
    ASSERT_NEAR(5, 6, 2); 
    // similar idea but you can specify the absolute difference 
    // such that two values are considered equal as the third arg

    ASSERT_ANY_THROW(throw std::string("hello"));

    ASSERT_THROW(throw std::runtime_error("Test"), std::runtime_error);

    ASSERT_NO_THROW([](){
        std::cout << "Hello";
    }());
}
```

Now what happens if a test fails? Well, we get a pretty helpful error message.

```C++
TEST(SuiteName, testName3) {
    ASSERT_EQ(300 - 24, 7859 * 5634);
}

```

```
../../Tests/Tests/main.cpp (<line number here>) expected equality of these values:
    300 - 24
        Which is: 276
    7859 * 5634
        Which is: 44277606
```

## Test Fixtures

A lot of times tests will need to share data or logic. In that case we can use a test fixture. A test fixture is a class derived from `testing::Test`. Any protected members will be available in all tests that use the fixture. It can provide a default constructor/destructor or override the members `SetUp()` and `TearDown()` to provide initialization and cleanup logic for any shared data. The constructor and `SetUp()` are called before each test and the destructor and `TearDown()` is called after each one. When I say "shared data", I don't mean that the actual instances of each member are shared, but rather each test has its own instance of the same members.

Internally, the `TEST` macro creates a class that is a subtype of `Test`. We can create a test fixture and use the macro `TEST_F` to create a test that subtypes this custom fixture. The first argument to `TEST_F` is the test fixture name and the second argument is the name of the test.

```C++
class RingbufferFixture : public testing::Test {
protected:
    std::unique_ptr<Ringbuffer<int>> r1, r2;

    void SetUp() override {
        r1 = std::make_unique<Ringbuffer<int>>(10);
        r2 = std::make_unique<Ringbuffer<int>>(10);

        r2.push_back(10);
        r2.push_back(20);
        r2.push_back(30);
    }

    void r2Add(std::initializer_list<int> elems) {
        for (e : elems) {
            r2.push_back(e);
        }
    }
};

TEST_F(RingbufferFixture, startEmpty) {
    ASSERT_TRUE(r1.empty());
}

TEST_F(RingbufferFixture, pushChangesSize) {
    ASSERT_EQ(r2.size(), 3);
}

TEST_F(RingbufferFixture, maxSizeUpheld) {
    r2Add({1, 2, 3, 4, 4, 5, 5, 7, 8, 9});
    ASSERT_EQ(r2.size(), 10);
}

TEST_F(RingbufferFixture, basicIndex) {
    ASSERT_EQ(r2[0], 10);
    ASSERT_EQ(r2[1], 20);
    ASSERT_EQ(r2[2], 30);
    r2Add({40, 50, 60});
    ASSERT_EQ(r2[3], 40);
    ASSERT_EQ(r2[4], 50);
    ASSERT_EQ(r2[5], 60);
}

TEST_F(RingbufferFixture, loopingTest) {
    ASSERT_EQ(r2[0], 10);
    ASSERT_EQ(r2[1], 20);
    ASSERT_EQ(r2[2], 30);
    r2Add({40, 50, 60, 70, 80, 90, 100, 200, 300});
    ASSERT_EQ(r2[0], 100);
    ASSERT_EQ(r2[1], 200);
    ASSERT_EQ(r2[2], 300);
}
```

## Parameterized Tests

Right now, if we wanted to do randomized or repeated testing on different inputs, we'd need to use loops in a single test. That's not too bad, but it can get hard to debug what input caused the test to fail. More annoyingly, it would require us to write a loop in all test cases we'd want to repeat on different inputs. Luckily, we can use parameterized tests. First, we create a test fixture which inherits from `::testing::TestWithParam<T>` where `T` is the type of the input. If we need multiple inputs we can make `T` a tuple. Then we define our tests similar to the test fixture but with the `TEST_P` macro.

```C++
class ParameterizedRBFixture : public ::testing::TestWithParam<int> {
protected:
    const int startingSize = 10;
    Ringbuffer<int> rb{startingSize};

};

TEST_P(ParameterizedRBFixture, pushTest) {
    const int param = GetParam(); // get param to test
    rb.push_back(param);
    ASSERT_EQ(rb[0], param);
}

TEST_P(ParameterizedRBFixture, pushSizeTest) {
    const int param = GetParam(); 
    for(auto i = 0; i < param; ++i) {
        rb.push_back(i);
    }
    ASSERT_EQ(rb.size(), std::min(param, startingSize));
}


INSTANTIATE_TEST_CASE_P(RingBufferTests, ParameterizedRBFixture,
    testing::Values(1, 711, 1989, 2013));
// Calls ALL tests of the fixture with the specified values

// instead of specifying values by a parameter pack, we can also use
// testing::ValuesIn() which generates values from an array, container, or begin and end iterator

int vals[] = {4, 56, 70, 0, 100, 230};

INSTANTIATE_TEST_CASE_P(RingBufferTests, ParameterizedRBFixture,
    testing::ValuesIn(vals));


// Using an iterator for randomized testing:

class UniformIntGenerator {
    std::mt19937 generator {std::random_device{}()};
    std::uniform_int_distribution<int> random{0, 2};
    uint64_t maxCount, curCount;
    constexpr static uint64_t finished = ~0;
    int val;
public:
    using value_type = int;
    using difference_type = size_t;
    using pointer = int*;
    using reference = int&;
    using iterator_category = std::forward_iterator_tag;

    UniformIntGenerator(uint64_t maxGenerations, int minVal, int maxVal) : random({minVal, maxVal}),
        maxCount(maxGenerations), curCount(0), val(random(generator)) {}

    /// Construct sentinel iterator
    UniformIntGenerator() : maxCount(0), curCount(finished), val(0) {}

    int operator*() const {
        if (curCount == finished) {
            throw std::out_of_range("Random generator exceeded generation limit");
        }
        return val;
    }

    UniformIntGenerator& operator++() {
        if (curCount + 1 < maxCount) {
            val = random(generator);
            ++curCount;
        } else {
            curCount = finished;
        }
        return *this;
    }

    UniformIntGenerator operator++(int) {
        const auto cpy = *this;
        ++(*this);
        return cpy;
    }

    bool operator==(const UniformIntGenerator& other) const {
        return curCount == other.curCount;
    }

    bool operator!=(const UniformIntGenerator &other) const {
        return !(*this == other);
    }
};

INSTANTIATE_TEST_CASE_P(RingBufferTests, ParameterizedRBFixture,
    testing::ValuesIn(UniformIntGenerator(100, -30, 30), UniformIntGenerator()));
// Use a begin and end iterator


```
Now instead of each single test looping over a list of values, we can make the entire test fixture take an argument and pass a series of parameters to the test fixture. I'd also like to mention that instead of specifying values, `testing::Range(start, end, [optional increment])` can be used instead of `testing::Values()` or `testing::ValuesIn()` to get a sequence of values starting at `start`, ending at `end`, and incrementing by 1 or `increment` each step.

If all you want to do is simply repeat tests, you can use the `--gtest_repeat` parameter to simply call the tests over again.

## Typed Tests

Typed tests are quite similar, expect they allow you to make a test fixture a template and specify the template arguments to repeat all tests with. A difference with typed tests however, is that we must define the test suite as typed as well using the macro `TYPED_TEST_SUITE` along with indicating a test should be types as well with the `TYPED_TEST` macro.

```C++
template<typename T>
class RBFix : public testing::Test {
protected:
    const int size = 10;
    Ringbuffer<T> rb{size};
};

using RBTestTypes = testing::Types<char, int, void*, long long>; // type alias is needed here
TYPED_TEST_SUITE(RBFix, RBTestTypes); // takes test fixture and types to instantiate it on

TYPED_TEST(RBFix, emptyTest) {
    ASSERT_TRUE(rb.empty());
}

// all tests will repeat for each type

```

We also can have typed parameterized tests by making the fixture a template and inheriting from `testing::TestWithParams<>`. Each test case would then use the `TYPED_TEST_P` macro. These types of tests would need to be registered and instantiated with macros.

## Matchers

GTest provides the generalized assertion `ASSERT_THAT` and `EXPECT_THAT`. The first argument is an object, and the second object is a matcher. Matchers are objects which ensure that a certain condition is met. There are matchers for containers, strings, numbers, etc. Matchers are part of GMock, and can be included with `<gmock/gmock.h>`. To link gmock, add `PRIVATE GTest::gmock` to the `target_link_libraries` call in CMake.

Matchers can also be composed into larger ones using `testing::AllOf()`, `testing::AnyOf()`, `testing::Not()` and `testing::Conditional(cond, m1, m2)`. The conditional uses `m1` if the condition is true, otherwise `m2`. Here are some examples, although I suggest taking a look at the documentation [here](https://google.github.io/googletest/reference/matchers.html)

```C++
using namespace testing;
ASSERT_THAT(num, AllOf(Gt(10), Lt(40))); // 10 < num < 40

ASSERT_THAT(str, StartsWith("Hello")); 
ASSERT_THAT(str, MatchesRegex("Id: [0-9]+"));
ASSERT_THAT(str, StrCaseEq("hElLo")); // equals ignore case
ASSERT_THAT(str, ContainsRegex("^Name")); // whole string does not have to match
ASSERT_THAT(str, AnyOf(EndsWith("goodbye"), HasSubstr("hello"))); 
// str.find("hello") != std::string::npos || str.find("goodbye") == str.size() - strlen("goodbye")

ASSERT_THAT(vec, Contains(100));
ASSERT_THAT(vec, ContainerEq(vec2)); // same as ASSERT_EQ but provides more informative error message

ASSERT_THAT(vec, ElementsAre(e0, e1, e2));
ASSERT_THAT(vec, WhenSorted(ElementsAre(e0, e1, e2))); 
// sorts the container first using operator<, then applies the matcher (in this case ElementsAre)
// in this case we sort the vector, then check elements

ASSERT_THAT(vec, WhenSortedBy([](auto a, auto b) { return a > b}, ElementsAreArray(vec2)));
// sorts by a comparator, then applies matcher
// in this case when check that the elements are the same as another container
// (can be an array, container, or begin and end iterators)

ASSERT_THAT(vec, UnorderedElementsAre(e0, e1, e2));
ASSERT_THAT(vec, UnorderedElementsAreArray(list.begin(), list.end()));
ASSERT_THAT(vec, UnorderedElementsAreArray(list));

ASSERT_THAT(vec, IsSubsetOf(vec3));
ASSERT_THAT(vec, IsSuperSetOf(e0, e2, e5));
// both IsSubset and IsSuperSet can take a container, comma separated elements, begin and end iterators, 
// or a begin iterator and size
// in both of these matchers, order doesn't matter

ASSERT_THAT(tuple, FieldsAre(42, "hi"));

struct S {
    int age;

    bool canRide() const {
        return age >= 13;
    }
}

ASSERT_THAT(object, Field(&S::age, Le(18))); // s.age <= 18
// ensures that an object has a given member
// and that member satisfies a matcher

ASSERT_THAT(object, Property(&S::canRide, 
    Conditional(object.age >= 13, Eq(true), Eq(false))));
// ensures that an object has the given no parameter const member function which, 
// when called, returns a value that matches the given matcher

ASSERT_THAT(ptrToVec, Pointee(SizeIs(Gt(10))));
// matches what the pointer points to with the given matcher
// ptrToVec->size() > 10
```

You might also find matchers useful as predicates for STL functions. This can be done using the `Matches()` function:

```C++
std::vector v = {10, 20, 30, 450};
auto it = std::find(v.begin(), v.end(), Matches(AllOf(Gt(10), Lt(100))));
```

Likewise, you might find that you want to use a predicate as a matcher. This can be done by wrapping the predicate inside the `Truly()` function.

```C++
const auto isOdd = [](int a) { return a % 2; };
// predicate just needs to return something implicitly convertible to bool

ASSERT_THAT(num, Not(Truly(isOdd)));
// assert num is even
```