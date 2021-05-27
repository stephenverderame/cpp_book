# Making a Guessing Game

I think that a guessing game is the stereotypical first thing to make after Hello World, so let's do that. To explore some more CMake features we'll be building off the previous "project". In the same directory as `main.cpp`, add a new folder called `guessing_game` and in it create a `game.cpp` file and another `CMakeLists.txt` file. So you're directory should look something like this:

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

In the top level directory we want to tell cmake to include a subdirectory in the project. We do this with `add_subdirectory()`. Therefore, we must add the following line to our top level CMakeLists file:

`add_subdirectory(guessing_game)`

Variables have scope in CMake like a programming language. Thus, in the guessing game CMakeLists we don't need to set the C++ version again. All we need to have is the following:

```cmake
add_executable(guessing_game game.cpp)
```

Let's first start off with some basic starter code

```c++
#include<iostream>

int main() {

    std::cout << "Guess a number:" << std::endl;
    return 0;
}
```

This is basically just a glorified Hello World, but let's make sure our project was setup correctly. Running `cmake -B out` again and then `make` should result in a new folder being generated in the out folder called guessing_game with an executable inside it with the same name.