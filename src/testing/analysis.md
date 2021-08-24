# Static Analysis

Static analysis tools look at the source code and warn you of any possible errors. Some static analysis tools also warns about code
that doesn't adhere to certain guidelines such as the core guidelines.

`cppcheck` is a popular static analysis tool. It focuses on issues that might cause bugs, however I believe it
(at least the GUI version) now includes the checks done by `clang-tidy`, which is another
static analysis tool that enforces the Cpp Core Guidelines.

Unix users can simply download it from your package manager. It has a command line and GUI interface.
Source code and Windows installer is available [here](http://cppcheck.sourceforge.net/).
On Windows, make sure to add `C:/ProgramFiles/CppCheck` (or whatever its download location is), to your PATH
so that you can use the `cppcheck` in the command line.

For windows users, I would suggest the cppcheck GUI as it's very easy to use. Visual Studio
also has a cppcheck plugin, however it seems to only work for Visual Studio projects (not CMake).

Some helpful command line arguments:
* `-i [directories]` - search through the following directories
* `--library=<lib>` - uses information about an external library such as `googletest` or `openssl`

More information can be found online.