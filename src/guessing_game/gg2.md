# Coding the Guessing Game

Don't worry about understanding everything right now, all these features will be explained in more detail later. I just want you to get a feel for C++ right now.

## Generating a Random Number

Let's write a function to generate a random number. First include the `<random>` header. 

```c++
auto getRandNumBetween(int min, int max) {
    std::random_device rd;
    std::mt19937 generator(rd()); //rd() computes a seed for the mt19937 generator
    std::uniform_int_distribution distrib(min, max);
    return distrib(generator); // computes a random number between min and max
}
```

`std::random_device` is in it of itself, a random number generator of sorts. It overloads `operator()` to allow the parenthesis syntax `rd()` which is the same for calling a function. `random_device` is known as a callable object because it overloads `operator()`. We use this generated number to seed a Mersenne Twister random number generator by passing the computed seed from `rd` to the constructor of `generator`. A constructor is a function used to initialize an object. `generator` is then used as an argument to `operator()` of the uniform integer distribution between `min` and `max`. Once again the uniform distribution is a callable object, something we'll talk more about later, but unlike `rd`, its `operator()` takes an argument. `min` and `max` are passed as arguments to the constructor of `uniform_int_distribution`. 

Unlike `main`, this function is declared to return `auto`. `auto` is not the return type for the function, instead it's indicating that the compiler should figure out what the return type is. In this case it's `int`.

I'll admit, that for something simple like this, I would normally use C's `rand()` function. This function always returns an integer between `0` and `RAND_MAX`. So an alternative implementation would be

```c++
auto getRandNumBetween(int min, int max) {
    srand(clock()); //seed the random number generator with the current time
    return rand() % (max - min) + min;
}
```

I'll stick to the previous implementation for the game. I should also note that you should not use `rand()` for something that relies on randomness. [^1]

## Getting User Input

Let's create another function to get input from the user.

```c++
auto getUserGuess() {
    auto guess = 0;
    std::cin >> guess;
    return guess;
}
```

We first initialize a new variable called `guess`. We once again use type deduction and give this variable `auto` as its type. We'll cover this more later, but you should generally always use `auto`. It might seem like an extra letter to type now (as opposed to `int` which is the type the compiler will infer `guess` to be) but it will save a bunch of headaches later. Next we parse whatever the user types in standard input as an integer, and store the result into `guess`. Finally we return `guess`.

Now what if a user doesn't type a number? Well, the way numbers are parsed right now, any non numeric input will just be converted to `0`. This is not ideal. So instead, let's implement the function another way. We'll now need the `<string>` header for this:

```c++
auto getUserGuess() {
    std::string guessStr;
    std::getline(std::cin, guessStr);
    return std::stoi(guessStr);
}
```
Here, we use `std::getline` to get a line from `std::cin` and store the result in `guessStr`. `guessStr` is an output parameter and you should avoid writing functions with them.
Now what do we get when we type something that's not a number? Well we get:

> terminate called after throwing an instance of 'std::invalid_argument'
>
>   what(): stoi

What's going on is that `stoi` has thrown an exception, `std::invalid_argument`. Any exception thrown out of `main` terminates the program. This might seem worse, but we can improve upon this by catching the exception. Let's factor out the exception handling code and make a helper function.


```c++
auto getUserInput() {
    std::string guessStr;
    std::getline(std::cin, guessStr);
    return std::stoi(guessStr);
}

auto printNonNumMsg() {
    std::cout << "You may only guess integers." << std::endl;
    std::cout << "Please guess again:" << std::endl;
}

auto getUserGuess() {
    try {
        return getUserInput();
    } catch (const std::invalid_argument & exn) {
        printNonNumMsg();
        return getUserGuess();
    }
}

```
`getUserInput()` is what previously was `getUserGuess()`. Now `getUserGuess()` tries to get a number from the user, and if that fails with `std::invalid_argument` it will print an error message and recurse, trying again. The catch block catches `invalid_argument` by *reference* which is denoted by `&`. So instead of the exception being copied, only a reference to the exception is copied. We declare this reference `const` so we can't mutate it. This is good practice and you should make everything that can be const, const. Passing by reference also allows this catch block to behave *polymorphically* and accept any subtypes of `std::invalid_exception`(which there are none in the standard library, but it's always good to catch exceptions by reference for this reason)

## Wiring it Up

Let's return to our `main()` function to finish the game. First we'll get a random number and store it in a constant. Then we'll loop until the user's guess equals that number. Since we won't be updating the range of random numbers generating during runtime, it's a good idea to make `constexpr` "variables" which are passed to `getRandNumBetween()`. This means that not only are the values constant, but they're also available at compile time. Therefore, in the compiled code they don't take up space, instead the compiler just inserts the literal values into the code whenever they are used.

```c++

constexpr auto min_value = 0;
constexpr auto max_value = 100;

int main() {
    const auto secretNum = getRandNumBetween(min_value, max_value);
    auto guess = 0;
    do {
        std::cout << "Guess a number between " 
            << min_value << " and " << max_value << std::endl;
        guess = getUserGuess();

        if (guess < secretNum) {
            std::cout << "Too low!" << std::endl;
        } else if (guess > secretNum) {
            std::cout << "Too high!" << std::endl; 
        }
    } while (guess != secretNum);
}

```

Here, we use a do-while loop, which  is like a while loop except it always performs at least one iteration. They can be harder to reason about and are somewhat controversial. Notice we also declare `secretNum` constant since it cannot change. I tend to use `snake_case` for compile time constants, `camelCase` for functions and variables, and `CapitalizedCamelCase` for classes, structs, enums, and unions. `AHH_ITS_A_SNAKE_CASE` is reserved for macros and should not be used for anything else.

We can use Uncle Bob's *step-down rule* and clean this up a bit by factoring out the prints into other function. Basically, his idea is that a function should have only one level of abstraction. This one has a few: the high level `getUserGuess()` and `getRandNumBetween()` and the low level `cout`s. So here's the final code:


## Final Code

```c++
#include <iostream>
#include <random>
#include <string>

constexpr auto min_value = 0;
constexpr auto max_value = 100;

auto getRandNumBetween(int min, int max) {
    std::random_device rd;
    std::mt19937 generator(rd());
    std::uniform_int_distribution distrib(min, max);
    return distrib(generator);
}

/**
 * @return the number input by the user
 * @throw std::invalid_argument if the user doesn't type a number
 */
auto getUserInput() {
    std::string guessStr;
    std::getline(std::cin, guessStr);
    return std::stoi(guessStr);
}

auto printNonNumMsg() {
    std::cout << "You may only guess integers." << std::endl;
    std::cout << "Please guess again:" << std::endl;
}
/**
 * Prompts the user for an integer
 */
auto promptGuess() {
    std::cout << "Guess a number between " 
        << min_value << " and " << max_value << std::endl;
}
/**
 * Gets a user's guess. Continues to prompt for a number if the
 * user does not type a number
 */
auto getUserGuess() {
    try {
        promptGuess();
        return getUserInput();
    } catch (const std::invalid_argument & exn) {
        printNonNumMsg();
        return getUserGuess();
    }
}
/**
 * Displays a hint based on the user guess
 * @param actual the real number
 */
auto displayGuessHint(int guess, int actual) {
    if (guess < actual) {
        std::cout << "Too low!" << std::endl;
    } else if (guess > actual) {
        std::cout << "Too high!" << std::endl; 
    }
}

/**
 * Tells the user they guessed num correctly
 * @param num the correct number
 * @param tries the amount of tries to guess correctly
 */
auto displayWin(int num, int tries) {
    std::cout << "Congrats! You guessed " << num 
        << " correctly in " << tries << " tries!\n";
}

int main() {
    const auto secretNum = getRandNumBetween(min_value, max_value);
    auto guess = 0;
    auto tries = 0;
    do {
        ++tries;
        guess = getUserGuess();
        displayGuessHint(guess, secretNum);
    } while (guess != secretNum);
    displayWin(secretNum, tries);
}
```

One thing to note is that we define all of our functions before they are used. This is necessary in C++ while not something done in Java. It's good practice to put the function definitions close to where they are going to be called. It's also good to keep functions within about 80 characters wide and 20 lines long.

Is this over-engineered? Probably. But I really think it's beneficial to start good habits now instead of having to unlearn bad ones later.

---

[^1]: Something like a Monte Carlo simulation or an algorithm that relies on randomization to be efficient.