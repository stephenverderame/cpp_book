# More on Move

I mentioned this earlier, but `std::move` doesn't move anything. Instead, it `casts` whatever you pass to it to an `rvalue`. Since an rvalue indicates giving up ownership, I manual `std::move()` says "hey, I'm done with this and you can do whatever you want with it". Therefore, you should **never** use a value after you have manually moved it. 

As I mentioned, the idea of moving is to swap the internals. For example, with `std::vector` that ends up copying the internal pointer and size from one vector to the other and invalidating both values in the original object. The original vector would then just be an empty shell waiting to be destroyed.
Here's a rough example in our Socket RAII class.

```C++
class Socket {
private:
    unsigned int sock;
public:
    Socket() : sock(socket(AF_INET, SOCK_STREAM, 0)) {}
    ~Socket() {
        if(sock != INVALID_SOCKET)
            closesocket(sock);
    }
    Socket(Socket&& other) : sock(INVALID_SOCKET) {
        std::swap(sock, other.sock);
    }
    Socket& operator=(Socket&& other) {
        std::swap(sock, other.sock);
        return *this;
    }
};


```

Many classes express things that don't make sense to be copied. (For example our socket class, or a thread). However it still makes sense to transfer ownership of them. That's where moves come in. Let's say you want a vector of Sockets to keep track of all the clients connected to a server via TCP. The following will not compile

```C++
std::vector<Socket> sockets;

Socket s;
sockets.push_back(s); //bad, tries to copy
sockets[userId] = s; //bad, tries to copy
```
```C++
std::vector<Socket> sockets;

Socket s;
sockets.emplace_back(Socket()); //emplace_xxx takes an rvalue
sockets[userId] = std::move(s); //invokes move constructor
// cannot reference s anymore
```

It's pretty dangerous to use std::move() to manually move an lvalue. If you do it, it should be at the end of the lvalue's scope, so that it can't be accidentally referenced later.

```C++
std::unique_ptr<Port> cmr::addPortDecorator(
    std::unique_ptr<Port>&& port, 
	std::shared_ptr<msgFmt::MsgFormatter> fmt,
	/*enum*/ PortDecorator d)
{
	// only one PortDecorator so I don't use a switch here
	return std::make_unique<ThreadPort>(std::move(port), fmt);
    // Where ThreadPort constructor takes an rvalue reference of a port unique_ptr
}
```
Here port binds to an rvalue, so no copy is made coming in. Port itself is an lvalue (that binds to rvalues) and thus we must use `std::move()` to pass along the rvalue which was passed to `addPortDecorator()`. Notice also the move is at the last place we reference port. 
