# Classes

C++ provides the access modifiers `private`, `public`, and `protected`. Unlike Java, these modifiers define sections of definitions that belong to that modifier instead of having to specify it for each member. Public members are part of the class's public interface and accessible to everyone, `private` is accessible to itself and friends only and `protected` is accessible to itself, friends, and subclasses. A class's members are `private` by default.

The class scope begins at the opening `{` after the class name and extends to the closing one. Members declared in this space are part of the class scope, and instances of them are destroyed when their owning instance object is destroyed. A special function known as a destructor is called when the owning instance is about to be destroyed, and similarly a constructor is called to create an instance of the class. 

```C++
class Person {
    const int ssn; //private member
public:
    // public members
    std::string name;
    int age;
protected:
    std::string homeAddress;

public:
    Person(const std::string & name) {
        ssn = 0;
        this->name = name;
        age = 0;

    }
    ~Person() = default;
    // destructor, we don't need anything special for this class
    // so we set it equal to default to use the default implementation
    // we didn't even have to declare the destructor in this case,
    // the compiler would generate one with the behavior we need

    Person(const std::string & name, int age, const std::string & addr) :
        ssn(0), name(name), age(age), homeAddress(addr) {}
    // This is a constructor initialization list
    // its more efficient then setting the values in the actual
    // constructor
};


Person p1("Harry");
Person p2{"Harry"};

const auto p3 = Person("Joe", 20, "500 Research Drive");
const auto name3 = p3.name;
const auto p4 = Person{"Jimmy"};

Person p5 = "Kyle";

```

Like any other function, we can provide overloads for constructors so that we can construct the object with different amounts/types of parameters. A constructor can have a constructor *initialization list*. This is done by specifying the initial values of member variables following a colon using function syntax. The advantage of this is that you only set the variable once. Otherwise, the members are initialized, then the constructor is called which may assign them to a different value.

Members values are initialized top-to-bottom, left-to-right, and the order of initialization in the initialization list must match. Therefore, in this example I could not set the value of `ssn` after I set the value of `name` because `ssn` is above `name` and therefore is initialized first. The reason C++ handles initialization this way is so that if initialization of an object fails and only some members were initialized, we can safely destroy the members that we did get a chance to initialize. The destruction occurs in reverse order of initialization.

Notice that `ssn` is a `const` member yet I'm still able to "change" its value. `const` members may be set once and only once, and that can occur only in the constructor initialization list or where they are declared if they are `inline` members (more on this later).

Finally, I'd also like to point out the constructor that takes one argument. Constructors that take one argument allow implicit conversion from the type of that argument to the class as demonstrated by the line `Person p5 = "Kyle";`. If this is not the semantics you intend, you should declare the constructor `explicit` like so:

```C++
    explicit Person(std::string & name) //..
```

Another example: (not complete)

```C++
class Socket {
    unsigned sock;
public:
    Socket() : sock(socket(AF_INET, SOCK_STREAM, 0)) {
        std::cout << "Socket initialized!\n";
    }

    ~Socket() {
        closesocket(sock);
        std::cout << "Socket destroyed\n";
    }

    bool writeAll(const std::string & msg) const {
        unsigned written = 0;
        do {
            auto sent = send(sock, &msg[written], msg.size() - written);
            if (sent < 0) break;
            written += sent;
        } while (written < msg.size());
    }
};

{
    const Socket sock; // Socket initialized
    sock.writeAll("Hello world");
} // Socket destroyed
```

Since `writeAll` does not modify any state of the instance, we can declare it `const`. Methods that are declared `const` can be used by instances declared `const`, however non-const methods cannot be used since they are allowed to modify state which would break the constness of the instance. The socket API is written in C, so technically `sock` is a handle which refers to a buffer in the OS that *is* mutating. But from our perspective we can say that `writeAll` is `const` since it doesn't change the values of members of an instance of `Socket`.

Also notice how the destructor is called when the object goes out of scope. This allows us to ensure our resources are freed when we're done using them. In this case the destructor cleans up our resources by closing the socket that was created by the constructor. Failing to do this would result in a *resource leak*.

As you see, we can use the dot operator (`.`) to access public members of an object with an automatic lifetime. 

I'll dedicate an entire chapter to this topic, so we'll hit this again later.