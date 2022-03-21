# Getting Started

First, let's get all of our tools setup. The most out of the box setup would probably be the Visual Studio IDE. 
However, for this book I'll be using tools that are typical in a Systems programming setting. Windows users might want to consider setting up [WSL](https://docs.microsoft.com/en-us/windows/wsl/install-win10) 
or using [Cygwin](https://www.cygwin.com/)

In any case, the first thing you'll need is a C++ compiler such as g++ and while we're at it, CMake as well. 
Linux users may simply run `sudo apt install g++ cmake -y`. Furthermore, we'll want a text editor. I would suggest Visual Studio Code. 
If you want to be hardcore you can also use Vim or anything else you would prefer. 
If you use Visual Studio (the IDE not VS Code), MSVC and CMake are installed when you select the C++ Desktop Development package in the installer.

You're more than welcome to use an IDE, but I will be giving instructions for operating on the command line. 
The GUIs and IDEs should be rather self-explanatory though. Due to the nature of working on embedded systems, I would suggest getting used to using the command line one way or another. 
You never know when the team laptop's UI dies...

C++ has a Standards Committee which agrees upon the specifications of the language. 
From there different vendors such as GNU and Microsoft implement their own compilers which adhere to the language specification. 
Furthermore, since 2011 the Standards Committee has agreed to release a new specification every 3 years. 
For this book we will be using C++17, and I will talk about a few points in C++20, but I will explicitly note when topics only apply to C++20. 

So, enough introduction! Let's get started!