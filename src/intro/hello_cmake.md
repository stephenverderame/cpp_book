# Hello CMake

We'll talk about CMake more in-depth later, but very quickly, what CMake does is abstract away the compiler from the build process. 
In the previous section, we explicitly invoked g++. But what if we have clang or msvc? That's where CMake comes in. 

All CMake projects require a top-level `CMakeLists.txt` file which is used to generate the build files and invoke the compiler. 
CMake makes it easy when we start to develop projects with many source files and when we want applications to be cross-platform.

In the same directory we put `main.cpp`, let's create a new `CMakeLists.txt` file. 

```cmake
cmake_minimum_required(VERSION 3.8)
project(hello_world)
set(CMAKE_CXX_STANDARD 17)

add_executable(hello main.cpp)
```

`cmake_minimum_required` sets the minimum cmake version that must be used to build the project. `project(hello_world)` creates a new Cmake project called hello_world. 
`set(CMAKE_CXX_STANDARD 17)` sets the C++ version to C++17. `CMAKE_CXX_STANDARD` is a variable, and `set()` sets that variable to a value. Finally `add_executable(hello main.cpp)` creates a new executable from the source file `main.cpp` and calls it `hello`.

Notice how nothing here is specific to g++.

Next, run `cmake -B out` which will generate the build files in a directory called out. If you don't specify the output directory with the `-B` flag, the default output directory is the same directory as the 
CMakeLists.txt file. To build it, `cd` into the `out` folder and invoke `make`. 
This will invoke the compiler through CMake-generated Makefiles and compile the executable. Once again, this should produce a file called `hello`, which upon execution should result in:

> Hello World

For Windows: Cmake will use the Visual Studio generator and create project files which can be opened in Visual Studio and built like any other project. Therefore, do not invoke `make`.