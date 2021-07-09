# CPP Book Outline
0. About
1. Getting Started
    * Hello World
    * Hello CMake
    * Basic Control Flow? -> Show syntax of if's whiles, etc
2. Introduction by example (Guessing Game)
    * Always auto, const correctness, functions, control flow, templates (random)
    * Compile without warnings
3. Basics
    * Values and References
        * Type conversions
    * Variables
        * Naming variables
        * Scope
        * Const correctness
    * Control Flow and operators
        * Comma, bitwise
    * Comments
        * Specs, Javadocs, Documentation
    * Functions
        * Pass by Value vs Reference
        * Function overloading
        * Function design (step-down rule, one thing, naming, returning references, avoid out params, pure functions)
        * Constexpr
4. Tools and Testing
    * CMake and Catkin
    * Doxygen
        * Grouping
    * GTest
        * TDD
        * Humble Object
    * GMock
    * GDB and GProf
    * Clang-tidy
5. Using STL
    * Strings
    * Arrays
    * Iterator Usage
        * Enhanced for
    * Vectors
    * Maps
    * Deques and Lists
6. IO
    * fstream
    * stringstream
    * std::filesystem
7. Organization
    * Structs
    * Enums (scoped)
    * Namespaces
        * Name lookup
        * Using keyword
    * Classes
    * Encapsulation
    * Initialization Order
    * Unions and bit fields
8. OOP
    * Class design
        * Coherence, 
    * Constructor and Destructor
        * Public and nonvirtual or private and virtual
        * Delegating constructors
    * Operator overloading
    * Conversions and explicit conversions
    * Inheritance (public, private, multiple)
        * Object slicing
    * Virtual Functions
        * NVI Idiom
    * = delete and = default
    * static
        * dead reference problem
    * shadowing and using
9. RAII
10. Pointers
    * Smart Pointers
    * Const pointers and references
    * PIMPL and Encapsulation
    * C arrays
11. Exceptions
    * Exception guarantees
    * Noexcept
    * Being exception aware
12. Rvalues and Move semantics
    * RValues
    * Move
    * Move constructor
        * Move, copy, constructor generation
    * Swap and Copy-Swap Idiom
13. Templates
    * Partial and full template specialization
        * std::hash
    * Concepts
        * What your classes need to work with the STL
        * Type traits
14. Casting and RTTI
    * Static, const, and reinterpret cast
    * Dynamic Cast
    * Typeid and type_info
    * narrow cast
15. About the Compiler
    * Preprocessor
        * Macros
        * Include guard
    * Linking
    * Header and code files
    * Inline and constexpr
    * Switch (jump tables)
    * Optimizations
    * RVO and EBO
        * Copy elision
    * SBO
16. Type Deduction
    * Always Auto
    * Auto and decltype
17. Universal References
    * Forwarding
18. Iterators
    * Iterator categories
    * Creating your own
19. Functional Programming
    * Callable Objects
    * std::function
    * bind
    * lambdas
        * captures and closures
    * Generic Algorithms
        * Parallel policies
        * Heaps, sorting, searching
    * Tuple
    * Any
    * Variant
        * std::visit
    * Optional
20. Variadic Templates
    * Fold expressions
21. TMP and Template Design
    * Tag Dispatch
    * Int2Type
    * Unevaluated contexts
    * SFINAE
    * Typename and template keywords
    * Type Erasure
22. Policy Based Design
    * Template template parameters
    * Non-type template parameters
    * Case Insensitive String
    * Deleters
23. Template and Iterator Example: Rope
24. C++17 and More STL
    * Chrono and literals
    * Init if and constexpr if
    * Bitset
    * Regex
    * stringview
25. Concurrency
    * Take from prev book
    * Actors and messaging
26. Guidelines (excerpts)
27. Software Architecture
    * SOLID
    * Components, Modules, and Boundaries
    * ROS Nodes
    * Stable Abstractions
    * Clean Embedded
28. Design Patterns (excerpts)
    * Factory
    * Decorator
    * Observer
    * Visitor
    * Facade
    * Memento
    * Singleton and Dead References
29. C++20 (from prev book)
30. Examples
31. Project
32. Further reading
33. Possible Excercise Answers

Notes:
* ODR and how inline functions can be defined "multiple" times -> defined in headers