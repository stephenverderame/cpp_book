# Testing and Debugging

In this chapter we'll look at useful tools for testing and debugging. Besides the obvious usage of finding bugs, testing also forces us to program with an interface in mind, not an implementation. A lot of times, to write tests we'll be forced to create a test implementation and a real one. Thus, this forces us to create an interface in which both implementations can subtype. For example, when testing an HTTP client, it can be hard to write tests using a live connection to an HTTP server. First of all of the reading logic is incorrect, it becomes terribly difficult to even see the error responses from the server. Secondly, such error messages aren't going to be the most useful in tracking down bugs in your code. So, we can abstract away the live connection behind an interface, and have a live implementation and test implementation which might just write data to a buffer in memory.

A common misconception is that testing early and often is a tradeoff: more time spent now means less time debugging later and that we can streamline the development process, if necessary, by putting off tests until later. While it makes sense, it has been found that lack of testing takes more time in both the short and long term. Testing can guide our design process, identify oversights in the specification, help us learn new libraries, and more.

## TDD

TDD stands for test-driven development and it's the common methodology to write tests before you write the code that it's testing. The TDD programming cycle is as follows:
1. Create a failing unit test
2. Write code to make that unit test pass
3. Clean, refactor, and ensure it still passes
4. Repeat

Some advocate the following 3 laws of TDD, which lock you into cycles of roughly 30 to 60 seconds:
1. Do not write code until you write a failing unit test
2. Do not write more test code than is sufficient to fail. Not compiling is failing.
3. Do not write more production code than is necessary to make a failing test pass.

It's common for the amount of test code to rival or exceed the amount of production code. Furthermore, many advocate that test code should be just as clean and well-designed as production code. This is because that test code must be as easy to change as production code, the two code bases must grow and evolve together. If the test code is rigid, then they'll either have to be rewritten every time the production code changes (leaving you vulnerable to forgetting corner cases) or the rigidness of the test code will make you unwilling to make the necessary changes to production code. Ideally, the only different between test and production code is performance. For example, in production you might want to have overloads for passing and returning lvalue and rvalue references while during testing you might find it sufficient to simply pass by value. However, it's vital that test code be as readable and clean as production. Clean test code can even serve as documentation (in Rust, examples written in documentation comments, can be run by the test framework to ensure they pass).

Tests should be:
* Fast - quick and easy to run. One command or one button to run all tests. Get a quick result
* Independent - should not rely on any order of tests being run and not depend on other tests
* Repeatable - run in any environment on any machine
* Self-validating - have a single boolean (pass/fail) output
* Timely - should be able to write somewhat quickly before writing production code