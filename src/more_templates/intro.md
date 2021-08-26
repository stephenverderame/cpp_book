# Advanced Templates

In this chapter we'll look at some more generic programming techniques, along with some more ways we can run computations at compile time.

Many compilers have a *step-limit*, which roughly correlates to how many times they'll recurse before stopping compilation.
Compile-time computations are recursive, and this step-limit ensures that compilation doesn't take too long.
Most compilers have an option to increase the step-limit if you need to.

> "The keys to an engineer's heart are acronyms. Lots and lots of acronyms."
>
> \- Me, 2021