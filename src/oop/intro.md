## Recap of OOP Language

As this book is mostly focused on learning C++, I will not delve into the details, designs, and principles of any programming paradigm. However, to make sure we're all on the same page, here's a quick recap of some terms I throw around quite often without definition.

* **abstract class** - a class which does not contains its entire implementation.
* **abstract data type** - a module which defines a type
* **client** - a user of a type. Basically not the implementor.
* **concrete class** - a class that contains its implementation.
* **dynamic dispatch** - invoking a method based on the dynamic type of the method's owning object instance.
* **dynamic/actual/runtime type** - the type of the object at runtime.
* **encapsulation** - the principle of hiding data members or implementation details from clients that they don't need to know about.
* **inheritance** - language feature which enables a class to inherit the members of another. There is a technical different between this and subtyping which we'll discuss later, although sometimes I use them a bit interchangeably. 
* **interface** - the methods and members that a client can use to perform operations on a type.
* **mixin** - a class which subtypes a set of narrow interfaces which define properties of a type rather than a type itself.
* **module** - a discrete and independent unit of a program.
* **object composition** - composing an object of other subobjects. Essentially a class that has a member which is another class.
* **overload** - a function that differes from another in the type or number of parameters to provide different ways to invoke an operation.
* **override** - a method that (re)defines the implementation of a base class method.
* **polymorphism** - the principle of using any implementation of an interface where the interface is expected.
* **static/declared type** - the type that is known at compile time. You can think of this as the actual type name you write in your code.
* **subtype** - a type that implements the same interface as another.

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

> "Any problem in computer science can be solved with another level of indirection"
>
> \- David Wheeler