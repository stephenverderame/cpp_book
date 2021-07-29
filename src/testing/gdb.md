# GDB

GDB is the GNU Debugger for Linux. It allows us to use a command line interface to investigate the stack trace, set breakpoints, step through execution, and more. On Windows, Visual Studio comes with a debugger that is pretty self-explanatory to use.

In order to start debugging a program, we'll need to compile our program so that it includes the debugging symbol tables. To do this, we need to add the "-g" flag to the command line arguments of g++. In CMake, there are a few ways to do this. The first is to set the build type to `Debug` or `RelWithDebugInfo`. The latter produces an optimized release build except it also contains debug symbols. The former produces a debug build. Another option is to set C++ compiler flags. We'll discuss CMake more later.

```CMake
set(CMAKE_BUILD_TYPE Debug) # Sets the build type to debug

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g") 
# adds the -g flag to the c++ compiler options
# applies to all build targets

add_compile_options(-g)
# adds the -g to all targets in the current directory
# and all subdirectories

target_compile_options(TargetName PRIVATE "-g")
# sets the -g flag to the specific target
```

Now our compiled binary can be run with the debugger. With `gdb` installed, run the command `gdb <path to exe>` to start debugging. You'll enter an interactive CLI that accepts commands.

Without doing anything else but running the program, the debugger will break execution whenever the program crashes or an exception is thrown out of the program. To run the program use the `r` or `run` command specify any command line arguments the program may need.

A backtrace can be displayed with `backtrace` or `bt`. And `q` quits gdb.

It may be helpful to log output to a file. To enable logging use `set logging on`. You can change the output file with `set logging file <file>`. By default, output is displayed to terminal as well as being logged. You can change this by turning on redirect, which when enabled only outputs to the log file. This can be turned on with `set logging redirect [on/off]`. Disable logging with `set logging off`. Everything printed to the terminal, will be saved in the log file.

Here are a list of some commands:
* `b`/`break` `(<class>::<member function> | <line number> | <file>::<line number> | <function name> | +<relative line num>) [if <expr>]`
    * `b 45` - breakpoint at line 45 of main file
    * `b MyClass::doFoo` - breakpoint at start of `doFoo()` of class `MyClass`
    * `break main` - breakpoint at start of `main()`
    * `b +10` - breakpoint 10 lines down from current position
    * `b source.cpp::23 if x == 2` - breaks line 23 in source.cpp if a variable x is 2
* `info breakpoints` - get a list of breakpoints. Each one has a numeric id
* `delete <breakpoint number>`
* `ignore <breakpoint number>`
* `skip <breakpoint number>`
* `step`/`s` `[lines]` - continue to next line of source code. Optional param amount of lines 
* `next`/`n` `[lines]` - same as `step` but does not trace through (step into) functions
* `stepi`/`si` `[instructions]` - goes to next *machine instruction*. Option param amount of instructions. Similar to `step`
* `nexti`/`ni` `[instructions]` - same as `next` but with machine instructions.
* `c` - continues to next breakpoint or error
* `watch <expr>` - breaks when the value of `expr` changes
    * `watch str` - breaks every time the value of the variable changes
    * `watch str.size() > 10`
* `condition <breakpoint number> <expr>` - makes the specified breakpoint conditional. The breakpoint will now only break when `expr` is true


* `info frame` - prints information about the current stack frame
* `u` - goes up one frame
* `d` - goes down one frame
* `frame <n>` - navigates to the nth frame. n = 1 is the parent frame with larger n going higher and higher up.
* `info locals` - prints values of local variables
* `info args` - displays information of arguments to current function

* `print`/`p` `[/<format>] <variable>` - displays the specified variable. `format` is an optional parameter to control the type the variable is displayed as. 
    * `p /s name` - displays the variable `name` as a C string.
    * `print x` - displays `x`
* `set <variable> = <value>` - sets a variable
* `call <function>` - executes the function
    * `call strlen(myStr)` - returns length of C string variable named `myStr`
    * `call malloc_stats()`, `call mallinfo()`, `call malloc_info(0, stdout)` - displays allocation and memory usage info to console
* `display [/<format>] <variable>` - displays the value of `variable` after every step or pause. Optional format code to control how `variable` is displayed.
    * `undisplay <variable>`
* `x /<count><format><unit> <address>` - displays `count` amount of `unit` starting from `address` in the given format
    * Units include:
        * `b` - byte
        * `h` - half word (2 bytes)
        * `w` - word (4 bytes)
        * `g` - "giant" or double word (8 bytes)
    * `x/12xb <addr>` - display 12 bytes in hex at `addr`


The formats for commands like `print` and `x` include the following:
* `s` - treat as C string
* `c` - print integer as char
* `d` - signed decimal integer
* `f` - floating point
* `o` - integer, print in octal
* `t` - integer, display in binary
* `u` - unsigned integer
* `x` - integer, display in hex

More information can pretty easily be found online. Also check out the [GDB cheat sheet](https://darkdust.net/files/GDB%20Cheat%20Sheet.pdf).

# GProf

GProf is the GNU profiler. To use the profiler we must compile with the "-pg" compiler flag. See above for how to do that in CMake. From there we can simply run the program like normal. When execution is finished, a file called `gmon.out` will be generated in the build directory. Next, we can simply run the `gprof` tool on this executable which will display a lot of profiler information. So you probably want to redirect the output to a log file.

```
gprof [<options>] <path to exe> gmon.out [> <log file name>]
```

Two important things to look out in the output is the *flat profile* and *call graph*. The flat profile provides a list of functions organized by time spent in each and looks something like this:
```
Each sample counts as 0.01 seconds.
  %   cumulative   self              self     total           
 time   seconds   seconds    calls  ms/call  ms/call  name    
 33.34      0.02     0.02     7208     0.00     0.00  open
 16.67      0.03     0.01      244     0.04     0.12  offtime
 16.67      0.04     0.01        8     1.25     1.25  memccpy
 16.67      0.05     0.01        7     1.43     1.43  write
 16.67      0.06     0.01                             mcount
  0.00      0.06     0.00      236     0.00     0.00  tzset
  0.00      0.06     0.00      192     0.00     0.00  tolower
  0.00      0.06     0.00       47     0.00     0.00  strlen
  0.00      0.06     0.00       45     0.00     0.00  strchr
  0.00      0.06     0.00        1     0.00    50.00  main
  0.00      0.06     0.00        1     0.00     0.00  memcpy
  0.00      0.06     0.00        1     0.00    10.11  print
  0.00      0.06     0.00        1     0.00     0.00  profil
  0.00      0.06     0.00        1     0.00    50.00  report
```
[Source](https://ftp.gnu.org/old-gnu/Manuals/gprof-2.9.1/html_chapter/gprof_5.html#SEC11)

The cumulative seconds is the time spent in the current function and all functions above that once in the table. Self seconds is the time spent only in that function. Gprof works by taking samples at fixed intervals. The sampling interval is displayed before the flat profile. In this case samples are every `0.01` seconds. Functions that are not called, are not compiled, or take an insignificant amount of time may not appear in the profile. The function `mcount` is used internally to perform this timing analysis.

Next is the call graph which shows how much time was spent in a function and its children. The graph is divided into entries separated by horizontal bars and looks something like this:
```
granularity: each sample hit covers 2 byte(s) for 20.00% of 0.05 seconds

index % time    self  children    called     name
                                                 <spontaneous>
[1]    100.0    0.00    0.05                 start [1]
                0.00    0.05       1/1           main [2]
                0.00    0.00       1/2           on_exit [28]
                0.00    0.00       1/1           exit [59]
-----------------------------------------------
                0.00    0.05       1/1           start [1]
[2]    100.0    0.00    0.05       1         main [2]
                0.00    0.05       1/1           report [3]
-----------------------------------------------
                0.00    0.05       1/1           main [2]
[3]    100.0    0.00    0.05       1         report [3]
                0.00    0.03       8/8           timelocal [6]
                0.00    0.01       1/1           print [9]
                0.00    0.01       9/9           fgets [12]
                0.00    0.00      12/34          strncmp <cycle 1> [40]
                0.00    0.00       8/8           lookup [20]
                0.00    0.00       1/1           fopen [21]
                0.00    0.00       8/8           chewtime [24]
                0.00    0.00       8/16          skipspace [44]
-----------------------------------------------
[4]     59.8    0.01        0.02       8+472     <cycle 2 as a whole>	[4]
                0.01        0.02     244+260         offtime <cycle 2> [7]
                0.00        0.00     236+1           tzset <cycle 2> [26]
-----------------------------------------------
```

The primary line is the line for the function that the entry is analyzing. This line has the name of the function not on an indent. The direct caller of the function being analyzed is also included in the same entry.

