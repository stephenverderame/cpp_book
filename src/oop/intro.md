### The term 'object'

If you're used to Java, to you, 'object' probably means an instance of a class. 
Therefore, an int would, by Java terms, not really be considered an object. 
But in C++, the standard defines an 'object' as a "region of storage". 
Objects can consist of sub-objects, and every variable is an object. Objects take up one or more memory locations, except bit fields which are separate objects but share a memory location.

What is a bit field? Well, it's a member variable designated to take up a different amount of bits than the amount of bits typically used by an object of its type.

```C++
struct Child {
    unsigned age : 4; //0 to 16, unsigned on its own means unsigned int
    unsigned avgGrade : 7; // 0 to 128 (maybe grade only goes to 100)
    bool sex : 1;
}
// without padding, this struct takes up 2 bytes (12 bits, but we can't have 1.5 bytes)

// age, avgGrade, and sex are 3 object taking 1 memory location
// they are also subobjects of an instance of Child, which is itself an Object

Child c;
// c is an object
```

This information won't become important until later (concurrency), but I thought I'd mention it now to avoid confusion.