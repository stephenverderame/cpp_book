# Testing and Debugging

In this chapter we'll look at useful tools for testing and debugging. 
Besides the obvious usage of finding bugs, testing also forces us to program with an interface in mind, not an implementation. 
A lot of times, to write tests we'll be forced to create a test implementation and a real one. 
Thus, this forces us to create an interface in which both implementations can subtype. For example, when testing an HTTP client, it can be hard to write tests using a live connection to an HTTP server. 
First of all if the reading logic is incorrect, it becomes terribly difficult to even see the error responses from the server. 
Secondly, such error messages aren't going to be the most useful in tracking down bugs in your code. 
So, we can abstract away the live connection behind an interface, and have a live implementation and test implementation which might just write data to a buffer in memory.

A common misconception is that testing early and often is a tradeoff: more time spent now means less time debugging later and that we can streamline the development process, if necessary, 
by putting off tests until later. While it makes sense, Jason Gorman and Uncle Bob have found that lack of testing takes more time in both the short and long term. 
Testing can guide our design process, identify oversights in the specification, help us learn new libraries, and more.

## TDD

TDD stands for test-driven development, and it's the common unit testing methodology to write tests before you write the code that it's testing. The TDD programming cycle is as follows:
1. Create a failing unit test
2. Write code to make that unit test pass
3. Clean, refactor, and ensure it still passes
4. Repeat

Some advocate the following 3 laws of TDD, which lock you into cycles of roughly 30 to 60 seconds:
1. Do not write code until you write a failing unit test
2. Do not write more test code than is sufficient to fail. Not compiling is failing.
3. Do not write more production code than is necessary to make a failing test pass.

Sometimes the amount of test code to rival or exceed the amount of production code. 
Furthermore, many advocate that test code should be just as clean and well-designed as production code. 
This is because that test code must be as easy to change as production code, the two code bases must grow and evolve together. 
If the test code is rigid, then they'll either have to be rewritten every time the production code changes (leaving you vulnerable to forgetting corner cases), 
or the rigidness of the test code will make you unwilling to make the necessary changes to production code. 
Ideally, the only different between test and production code is performance. 
For example, in production you might want to have overloads for passing and returning lvalue and rvalue references while during testing you might find it sufficient to simply pass by value. 
However, it's vital that test code be as readable and clean as production. 
Clean test code can even serve as documentation (in Rust, examples written in documentation comments, can be run by the test framework to ensure they pass).

Tests should be:
* **F**ast - quick and easy to run. One command or one button to run all tests and get a quick result.
* **I**ndependent - should not rely on any order of tests being run and not depend on other tests
* **R**epeatable - run in any environment on any machine
* **S**elf-validating - have a single boolean (pass/fail) output
* **T**imely - should be able to write somewhat quickly before writing production code

To state TDD another way, the steps are basically: Red, Green, Refactor. 
First you make it Red: create a failing test. This is vital because you need to know your test doesn't always pass. 
It would get cumbersome to have tests of tests so this red step is essentially the manual testing of our test case. 
Then you make it Green: make the test pass as fast as possible. Embrace bad engineering, don't worry about design patterns in this step just figure out how to implement the feature in the simplest way possible. 
In this step the goal is to wrap your head around a correct solution. It's hard to do two things at once so trying to make something clean and correct at the same time can make things extremely difficult. 
Finally, refactor: you've already implemented a solution once, so you will have a better idea of what patterns and idioms will work to clean up the code. 
You'll be able to more easily organize the code since you already know what it's doing. This step should NOT lead you to write new tests.

You should test to an interface, not an implementation. 
In other words, only test the public API. The thing that should trigger writing a test case is not a new function or class, but rather a new requirement of the software, or a new behavior in an interface. 
Some people find that "Behavior Driven Development" is a better name than "Test Driven Development" for this reason. 
So for example, when developing a helper method, you most likely shouldn't start by writing a failing test case for it, the tests for methods the helper is used in cover it already. 
This isn't always the best approach however: you might not be sure if the helper method is correct, or you might not be sure how to go about implementing whatever algorithm you're trying to write. 
In this case we can drop down into a lower "gear" and use TDD for the implementation details. However, once the algorithm is implemented, these tests should be deleted because they rely on an 
implementation and thus make the implementation details rigid. Tests should not know about the implementation. 
This should not be confused with me saying "glass-box testing is bad", rather what I'm saying is you don't want to enforce any constraints on the implementation. 
You may very well pick corner cases to test based on the implementation and code coverage, but changing the implementation or refactoring should not make these tests fail.

Sometimes, you might find you have a clear idea of what you're trying to implement. 
In this case we can "shift" into a higher "gear" and streamline this red, green, refactor process. You might choose to go straight from red to refactor and make a clean solution as you are making a correct one. 
Sometimes you might find that what you're implementing is so simple, it doesn't even warrant TDD.

The "unit" in unit testing is often times a module, and although OOP languages realize modules as classes, this does not mean every class is a module. 
A module satisfies an abstraction, and can be composed of multiple classes. But even so, a module may not be a "unit". 
Kent Beck highlights that the "unit" in a unit test is the test itself. In this sense he means that tests must be independent of each other.

> "I'm not a great programmer; I'm just a good programmer with great habits."
>
> \- Kent Beck