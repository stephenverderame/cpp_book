# RAII

RAII is probably the most important idiom in C++. It stands for Resource Acquisition is Initialization, although it has more to do with destruction than initialization. 
Objects stored on the stack have automatic lifetimes and are automatically popped off the stack when they go out of scope. 
RAII is the idea of wrapping some resource in an object with an automated lifetime so you never have to worry about cleaning it up. Here's the basic idea:
```C++
class Socket {
private:
    unsigned int sock;
    bool open;
public:
    Socket() : sock(socket(AF_INET, SOCK_STREAM, 0)),
        open(false) {}
    ~Socket() {
        closesocket(sock);
    }
};
//...


//In some function
{
    Socket s;
    //...
} // socket closed when s is destroyed
```
So what's the advantage of RAII you ask? Well now you need not worry about `Socket` being cleaned up. 
If an exception is thrown or the variable goes out of scope, the resource is automatically released. 
If you ask me this is quite a lot nicer than try-catch-finally blocks or Java's try-with-resources.

When you have behavior that must run on every single execution path, think RAII! 
If you have to cleanup a resource or call a resource freeing function (something like `free`, `delete`, `cleanup`, etc.), think RAII.

```C++
class BlanketErrorLogger {
    std::optional<std::string> msg;
public:
    BlanketErrorLogger(const std::string & errorMsg) : msg(errorMsg) {}

    void succeed() { msg = std::nullopt; } // make the optional empty

    ~BlanketErrorLogger() {
        if (msg) {
            std::cerr << msg.value() << "\n";
        }
    }

};

// DISCLAIMER: not necessarily the actual Ws2 API
class WinsockContext {
    inline static unsigned ctxCount = 0;
public:
    WinsockContext() {
        if (ctxCount++ == 0) {
            WSAData data;
            WSAStartup(MAKEWORD(2, 1), &data);
        }
    }

    ~WinsockContext() {
        if (--ctxCount == 0) {
            WSACleanup();
        }
    }
};

int main(int argc, char ** argv) {
    WinsockContext wsa;
    std::stringstream ss;
    // argv[0] is the current filepath
    for(auto i = 1; i < argc; ++i){
        ss << argv[i] << " ";
    }
    BlanketErrorLogger err("Command: \"" + ss.str() + "\" failed!");
    //
    // Complex socket operations
    //
    err.succeed();
    return 0;
}
```

In this example, no matter what happens we cleanup the Windows Socket API. If execution stops before `err.succeed()` is called, then we'll also get a log message with the command line arguments that caused the error.

Now you may be wondering why `ctxCount` is defined as `inline` and `static`. 
Well a `static` variable means there is only one copy of the data. We can have static variables in functions and classes. 
Here, we define `ctxCount` static so that we retain a count of all instances of `WinsockContext`. 
This allows us to check in the destructor if the current instance being destroyed is the last existing instance, and if so cleans up WSA. 
This check allows us to nest instances of `WinsockContext` without accidentally cleaning up the resource while other code is still using it.

```C++
int amtOfCalls() {
    static int count = 0; //only assigned once
    return ++count;
}

amtOfCalls(); // 1
amtOfCalls(); // 2
amtOfCalls(); // 3
```

A static variable is initialized the first time the path of execution goes through the static variable's definition. 
It's destroyed when the program ends, however the order of de-initialization between static variables when the program terminates is undefined. 
Therefore, using a static variable in the destructor of a class which has a static instance can lead to problems. 
This issue commonly arises in Singletons, and as such is referred to as the Singleton Dead Reference problem, which we'll discuss later.

Ok so that's `static`. Then what's the deal with `inline`? 
Well let's say that `WinsockContext` was defined in a header, and we include this header in multiple source files. 
Then if `ctxCount` weren't inline then we'd have multiple definitions of `WinsockContext::ctxCount`. 
`inline` variables allow us to declare and define variables in a header file by telling the compiler that if multiple definitions are generated, they're all the same and to choose one of them. 
Without `inline`, we'd have to do something like this:

WinsockContext.h
```C++
class WinsockContext {
    static unsigned ctxCount;
public:
    WinsockContext() {
        if (ctxCount++ == 0) {
            WSAData data;
            WSAStartup(MAKEWORD(2, 1), &data);
        }
    }

    ~WinsockContext() {
        if (--ctxCount == 0) {
            WSACleanup();
        }
    }
};
```

WinsockContext.cpp
```C++
#include "WinsockContext.h"
static int WinsockContext::ctxCount = 0;
```

Just to ram RAII into your head a tad bit more, here's another example:

```C++
// DISCLAIMER: Not necessarily the actual stb_image or OpenGL API
// I'm just doing this from memory
struct LoadImg {
    stb_uchar* img;
    int width, height, channels;
    LoadTexture(const char * file_path) {
        img = stbi_load(file_path, &width, &height, &channels, NULL);
        if (!img) {
            throw std::runtime_error(std::string("Failed to load image at ") +
                file_path);
        }
    }

    ~LoadTexture() {
        if (img) {
            stbi_free(img);
        }
    }

    // copy operations could delete data we are still using
    // delete it
    LoadTexture(const LoadTexture &) = delete;
    LoadTexture& operator=(const LoadTexture &) = delete;
    LoadTexture(LoadTexture &&) = delete;
    LoadTexture& operator=(LoadTexture &&) = delete;
};

GLuint channelsToImgFmt(int channels) {
    switch(channels) {
        case 4:
            return GL_RGBA;
        case 3:
            return GL_RGB;
        case 1:
            return GL_RED;
        default:
            throw std::runtime_error(
                std::string("Invalid number of img channels: ") + channels);
    }
}

class Texture {
    GLuint tex;
public:
    Texture(const char * file) {
        LoadImg img(file);
        glGenTextures(1, &tex);
        glBindTexture(GL_TEXTURE_2D, tex);
        const auto fmt = channelsToImgFmt(img.channels);
        glTexImage2D(GL_TEXTURE_2D, 0, fmt, img.width, img.height, fmt, GL_BYTE, img.data);
        glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_LINEAR);
        glTexParameteri(GL_TEXTIRE_2D, GL_TEXTURE_MIN_FILTER, GL_LINEAR);
    }

    ~Texture() {
        if(tex != GL_INVALID) {
            glDeleteTextures(1, &tex);
        }
    }

    // copy operations could delete data we are still using
    // delete it
    Texture(const Texture &) = delete;
    Texture& operator=(const Texture &) = delete;
    Texture(Texture &&) = delete;
    Texture& operator=(Texture &&) = delete;
};
```

Now I mention that copy operations can delete data we are still using. 
How? Well suppose we copied `Texture` by copying `tex`. 
`tex` is just a handle to memory allocated in VRAM, so we're copying the handle but not the actual data. 
But now we have two instances of classes that claim to own the data and will delete it when they are destroyed. 
This isn't good! The second destructor to get called will delete invalid data! 
If we wanted, we could override the copy operations to perform deep copies, but for simplicity, it's easier to just prevent those operations from occurring.

However, we could easily make `Texture` moveable.

```C++
    Texture(Texture && other) {
        tex = other.tex;
        other.tex = GL_INVALID;
        // invalidate the temporary's data
        // so that when it's destroyed it doesn't delete our data
        // other is a temporary or value we don't care about, so it's safe to
        // invalidate its handle
    }

    Texture& operator=(Texture && other) {
        if(tex != GL_INVALID) {
            glDeleteTextures(1, &tex);
        }
        tex = other.tex;
        other.tex = GL_INVALID;
    }
```

You may be wondering: "If the program is going to terminate anyway, must we be so careful with releasing resources?". 
In today's day and age, no, probably not. The OS should be able to cleanup any resources that a program doesn't when it terminates. 
But that's only *when* it terminates. What if we decide to catch texture loading errors and then try again with a lower resolution image? 
Perhaps whatever image we are trying to load is nonessential, so if it fails to load we just ignore it. 
This might not be something we are currently doing at the time of writing the code, but maybe months or years down the line that might change. 

### Possible Exercises

1. Try writing a RAII class that prints to standard out during constructions and destruction. Ensure that proper resource deletion occurs when exceptions are thrown.
2. Can you make the previous RAII class work with copying?

