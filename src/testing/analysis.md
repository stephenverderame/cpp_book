# Static Analysis

Static analysis tools look at the source code and warn you of any possible errors. Some static analysis tools also warns about code
that doesn't adhere to certain guidelines such as the core guidelines.

`cppcheck` is a popular static analysis tool. It focuses on issues that might cause bugs, however it also 
has extensions to enforce coding style checks. `clang-tidy`, another
static analysis tool that enforces the Cpp Core Guidelines can also be used from `cppcheck`.

Unix users can simply download it from your package manager. It has a command line and GUI interface (`cppcheck-gui`).
Source code and Windows installer are available [here](http://cppcheck.sourceforge.net/).
On Windows, make sure to add `C:/ProgramFiles/CppCheck` (or whatever its download location is), to your PATH
so that you can use the `cppcheck` in the command line.

I would suggest the cppcheck GUI as it's very easy to use. Visual Studio
also has a cppcheck plugin, however it seems to only work for Visual Studio projects (not CMake).

Some helpful command line arguments:
* `-I <include directory>` - search through the following include directories
* `--library=<lib>` - uses information about an external library such as `googletest` or `openssl`
* `--addon=<addon>` - enable an addon such as `cert` which enables checks for CERT coding guidelines
* `--enable=<check>` - enables checks with the given name, such as `all`, which enables all checks
* `--platform=<type>` - sets the platform type (such as `unix64`, `win64`, `avr8`, etc.)
* `--std=<std>` - sets the standard version (ex. `c11`, `c89`, `c++11`, `c++17`, `c++03`). The current default is `c++20`.

For the following file structure:
```
Project
|
|___include
|   |
|
|___src
|   |
|
|___test
|   |
|
```
we can run `cppcheck` with the command
```
cppcheck -I Project/include/ Project/src/ Project/test/ --library=googletest --std=c++17
```

More information can be found online.