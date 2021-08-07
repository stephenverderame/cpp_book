# More on Move

I mentioned this earlier, but `std::move` doesn't move anything. Instead, it *casts* whatever you pass it to an `rvalue`. Since an rvalue indicates giving up ownership, a manual `std::move()` says "hey, I'm done with this and you can do whatever you want with it". Therefore, you shouldn't use a value after you have manually moved it. 

As I mentioned, the idea of moving is to swap the internals. For example, with `std::vector` that ends up copying the internal pointer and size from one vector to the other and invalidating both values in the original object. The original vector would then just be an empty shell waiting to be destroyed.
Here's another rough example from our Socket RAII class.

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

## RTTI

Ok this admittedly has little to do with move semantics but I didn't know where to put it. It better belongs in the casting section but I didn't want to give half an explanation since it helps to know about prvalues and glvalues.

RTTI stands for runtime type information and it's how `dyanmic_cast` is able to check the dynamic type during a cast. RTTI information is encapsulated within an `std::type_info` object which is part of the `<typeinfo>` header. This object is hashable with the `hash_code()` member function, comparable with `operator==` and `operator!=`, can be ordered with the `before()` member function and can also print out an implementation defined name representing the type with the `name()` member function. `std::type_info` is neither constructible nor copyable. If you want to use it as a key or put it in a container, you can wrap it in an `std::type_index` which holds a pointer to a `std::type_info` object and is therefore copy constructible and assignable. `std::type_info` objects have static lifetimes so their pointers should not be manually freed.

The `name()` member function should be used for debugging purposes only. This name is implementation defined and it may return a different string for the same type between different compilations or even different executions of the program. `name()` also does not distinguish between reference and non reference types.

A `std::type_info&` is returned by the `typeid()` operator. This operator takes an expression or a type. Top level qualifiers (`const` or `volatile`) and reference-ness is ignored; so `typeid(int) == typeid(const int) == typeid(int&)`. If passed an expression that is a *glvalue*, the expression is executed and the dynamic (behaves polymorphically) resultant type's `std::type_info` is returned. If the expression is a *prvalue*, it's not executed and the static type's `std::type_info` is returned because prvalues are not polymorphic. Dereferencing a `nullptr` within `typeid()` will throw `std::bad_typeid` if the pointer being dereferences is polymorphic. If it is not (and therefore dereferencing it can only result in one type), no exception is thrown.

```C++
const std::type_info& ti = typeid(std::cout << "Hello there\n"); 
// expression executed and prints (bc operator<< returns an ostream reference -> glvalue)
std::cout << ti.name() << std::endl; 
// implementation defined but probably something like
// std::basic_ostream<char, std::char_traits<char>>

std::vector<std::type_index> types;
types.push_back(ti); //type_index is constructible and copyable

struct HerType {
    virtual ~HerType() = default;
}

struct MyType : public HerType {
    MyType() {
        std::cout << "Hello World\n"
    }
}

std::cout << typeid(MyType()).name() << std::endl;
// only prints (most likely) "MyType"
// constructor not executed

MyType mt;
HeyType& ht = mt;
std::cout << typeid(ht).name() << std::endl;
// likely "MyType"
```
