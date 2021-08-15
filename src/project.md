# Project

To put all of this together, we're going to make an HTTP client and server utility. The program will be able to host web servers and connect to others as well. 
It will be a command line program that accepts the following commands:
* `-serve <directory>` - create a server with the specified root directory
* `-get <url>` - GET request
* `-post <url>` - POST request
* `-port <port>` - set the port number on the server or request
    * Overrides any port specified in the URL for a request
* `--<header> <value>` - set a HTTP header in a request
* `-content [-f] <text>` - specifies the content in a request or server. An optional `-f` command may follow indicating that the content is to be read from the specified file. If the content contains spaces, it must be enclosed with quotes. If content is specified for a server, this essentially specifies the `index.html`.
* `-ssl [-only] <certificate>` - makes the server use SSL using the specified certificate file. If the `-only` flag is set the server rejects plaintext
* `-o <file>` - stores the result of a request into the specified file

The program should be cross-compatible on Windows and Unix systems. For windows, you will have to link with `Ws2_32.lib` and most of the networking stuff will be in the `WinSock2.h` header. You may also need `WS2tcpip.h` (and others). For Unix, you may want to look at `sys/socket.h, sys/types.h, netinet/in.h, sys/select.h, fcntl.h, arpa/inet.h, sys/ioctl.h, unistd.h, netdb.h`. The socket API for both libraries are very similar, but there are a few slight differences: (these are not the only ones)

| Windows | Unix |
| --- | --- |
| `ioctlsocket(socket, long cmd, u_long* arg)` | `ioctl(int fd, long cmd, ...) (sort of equal but not quite)` |
| `closesocket(socket)` | `shutdown(sock, SHUT_RDWR); close(sock)` (the `shutdown()` isn't necessary) |
| `WSAGetLastError()` | `errno` |

Winsock requires manual initialization with `WSAStartup()` and `WSACleanup()`.

Start with whatever OS you natively have, then try to support the other afterwards. This is a good case for extracting the OS specific code into a policy which can be passed to the socket classes as a template argument.

Since both of these APIs are C APIs, most of their functions return error codes. It's important to make sure these errors are handled.

## Setup

[Starter code available here](https://github.com/stephenverderame/CppBookProject). You will also need to install OpenSSL. For windows, you can download pre-compiled binaries via an installer [here](https://slproweb.com/products/Win32OpenSSL.html). OpenSSL also have a list of some other reccomended third parties for getting the pre-compiled binaries [here](https://wiki.openssl.org/index.php/Binaries).

For Unix users, you can simply install `libssl-dev` and `openssl` from your package manager. You will also need to make sure you have CMake and Doxygen installed. Ubuntu users can simply install `cmake` and `doxygen` from `apt` (other distros might have the binaries available from the package manager, otherwise go the the link)  and Windows users can find installations for [cmake here](https://cmake.org/download/) and [doxygen here](https://www.doxygen.nl/download.html). You may need to install GraphViz (what Doxygen uses to generate inheritance graphs) separately [here](https://graphviz.org/download/). For windows you should add `dot` to your path system variable via the installer.

## Goals

* Gain experience with GTest, GMock, and CMake
* Practice with static and dynamic polymorphism in C++ and programming to an interface
* Practice designing template function and classes
* Use C++ Idioms, Clean Architecture and Design Patterns
* Familiarize yourself with the standard library
* Learn about socket programming
* Experience with Doxygen

## Tasks

* Implement the Socket Class
    * This class will adhere to the RemotePort concept (see `Port.h`)
    * It will take a policy class that handles code changes depending on the OS
    * Test your implementation in `SocketTest.cpp`
    * Use `SSLSocket` for guidance
    * For Windows: create a RAII class for initialization and cleanup of Winsock. You might want to make this a singleton
* Implement Exception class(es)
    * I crudely throw integers just to show which function gets the error code you'd want to look at, this isn't that helpful for displaying error messages or catching exceptions
    * Implement an exception class or exception hierarchy 
* Implement the HTTP request and response frames
    * User should be able to choose HTTP version (parameter to `compose` method or pass to constructors?)
* Implement the HTTP client
    * Should take ownership of a `Port` -> reads and writes requests on this port
    * Be able to send request Http Frames and receive response frames
    * User should not have to do any string formatting (except for any content encoding)
* Implement HTTP content and transfer encodings:
    * Need to support chunked transfer encoding, and url type data 
    * Design Ideas:
        * HTTP writer decorators:
            * A class that would subtype some kind of HTTP interface and be composed of an implementation of that interface
            * A high level `set_content` or `write` (member or overrided from interface) function would then set the content of the underlying HTTP writer following the specified format and all relevant headers
        * Transaction Scripts:
            * Non-member helper functions set the writer's content and relevant headers following the specified format
        * Encoding strategy:
            * A callable object or implementation of some kind of strategy interface is passed to a `write` function
    * See `ChunkedTest` and `QueryTest`
* Implement the HTTP server
    * Should take ownership of a `Port` -> reads and writes requests on this port
    * Root directory (where all the paths in the request are relative to) should be customizable
    * Handles `GET` and `POST` requests
    * Supports URL encoding
    * See `ChunkedTest` and `QueryTest`
* Implement the command line argument marshallers
    * Recognize command line arguments
* ~~Feel free to~~ add your own tests and expand upon the existing ones
* Tie it all together:
    * Create a website with a C++ backend:
        * More than 1 page
        * Something that accepts user input
        * Bonus:
            * CRUD (create, read, update, delete) operations on some "database"
            * Fetching images (or some other non-plaintext data) from the site
* (Optional) Implement `FdSet`
    * Not really necessary because HTTP is a stateless connection protocol
    * But if you needed to keep track of active connections this would be vital
* (Optional) Add IPv6 Support to `Address`
    * Make Address a template accepting a policy which controls:
        * type of the `addrData` member. (`sockaddr_in` vs `sockaddr_in6`)
        * family `AF_INET` vs `AF_INET6`
        * the `is_ip` member function
    * OR use runtime polymorphism and a factory function so that the correct Address class can be chosen at runtime based on the IP address passed to the factory function
* (Optional) Add JSON support
    * JSON object and array types which can be serialized from and into JSON text
    * Support for sending and receiving JSON data in the HTTP client and server

## Possibly Useful

* [Similar project I developed a few years ago](https://github.com/stephenverderame/webchat)
    * Neither implementation is the best designed, however WebLib2 is better
* [HTTP Reference on MSDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview)
* [C++ Regex Library](https://en.cppreference.com/w/cpp/regex)
* [C++ Filesystem Lib](https://en.cppreference.com/w/cpp/filesystem)
* [C++ Streams Lib](https://en.cppreference.com/w/cpp/io)
* [Microsoft Docs for Winsock](https://docs.microsoft.com/en-us/windows/win32/winsock/getting-started-with-winsock)
* [Man Pages for POSIX Functions](https://man7.org/linux/man-pages/man2/socket.2.html)

## Other Notes

You can use preprocessor conditions around the definition of `WIN32` to conditionally enable code depending on if using Windows or Unix. 
`WIN32` is a macro defined by the compiler when compiling on Windows (even Win64). In the CMake, I also define `WINDOWS` when the platform is Windows and `UNIX` when it is not.

Feel free to change the implementation or design of anything. Although there is a comment saying not to, you may change the Port interface so long as you make sure to have tests.

The current project structure is as follows:
```
<Project Directory>
|   CMakeLists.txt
|---HttpProject
|   |   CMakeLists.txt
|   |---include
|   |---src
|   |---test
|   |   |   CMakeLists.txt
|   |   |---data
|   |
|---external
|---out
|
```