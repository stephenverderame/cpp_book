# Project

To put all of this together, we're going to make an HTTP client and server utility. The program will be able to host web servers and connect to others as well. It will be a command line program that accepts the following commands:
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
| `closesocket(socket)` | `shutdown(sock, SHUT_RDWR); close(sock)` |
| `WSAGetLastError()` | `errno` |

Various windows functions require a reference to a size passed as an `int*` while the POSIX api uses an `unsigned int*`. Furthermore Winsock requires manual initialization with `WSAStartup()` and `WSACleanup()`.

Start with whatever OS you natively have, then try to support the other afterwards. This is a good case for extracting the OS specific code into a policy which can be passed to the socket classes as a template argument.

Since both of these APIs are C APIs, most of their functions return error codes. It's important to make sure these errors are handled.