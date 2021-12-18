# Making a Guessing Game

I think that a guessing game is the stereotypical first thing to make after Hello World, so let's do that. 
To explore some more CMake features we'll be building off the previous "project". 
In the same directory as `main.cpp`, add a new folder called `guessing_game` and create a `game.cpp` and another `CMakeLists.txt` file in that folder. 
So your directory should look something like this:

```
project
|   CMakeLists.txt
|   main.cpp
|
|---guessing_game
|   |   CMakeLists.txt
|   |   game.cpp
|   |
|---out

```

In the top-level directory we want to tell cmake to include a subdirectory in the project. 
We do this with `add_subdirectory()`, so add the following line to the top-level CMakeLists file:

`add_subdirectory(guessing_game)`

Variables have scope in CMake like a programming language (CMake is Turing Complete).
Thus, in the guessing game CMakeLists, we don't need to set the C++ version again. All we need to have is the following:

```cmake
add_executable(guessing_game game.cpp)
```
Let's also enable all warnings. We can do this with the following in the top-level `CMakeLists.txt`.

```cmake
if (MSVC) 
    add_compile_options(/W4 /WX)
else ()
    add_compile_options(-Wall -Werror)
endif ()
```

`add_compile_options` adds compiler flags to all targets, whereas `target_compile_options` adds them to a single target (such as a single executable). 
It's good practice to enable the highest warning level and treat all warnings as errors. The above snippet provides flags to do this depending on the compiler.

So this is the complete top-level CMakeLists.txt:

```cmake
cmake_minimum_required(VERSION 3.8)
project(hello_world)
set(CMAKE_CXX_STANDARD 17)

if (MSVC)
    add_compile_options(/W4 /WX)
else ()
    add_compile_options(-Wall -Werror)
endif ()

add_executable(hello main.cpp)

add_subdirectory(guessing_game)
```


Now let's add some basic starter code.

```c++
#include<iostream>

int main() {

    std::cout << "Guess a number:" << std::endl;
    return 0;
}
```

This is basically just a glorified Hello World, but let's make sure our project was setup correctly. 
Running `cmake -B out` again should generate a new folder `guessing_game` in the output directory.
Invoking `make` in the `out` directory should compile an executable `guessing_game` in the guessing_game folder.