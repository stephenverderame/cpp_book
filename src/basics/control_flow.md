# Control Flow

Here are some examples of control flow in C++

#### If 

```c++
if(/* condition */) {

}
```

```c++
if(/* condition */) {

} else {

}
```

```c++
if(/* condition */) {

} else if(/* condition */) {

} else {

}
```

#### For Loop

```c++
for(/* initializer */;/* guard */;/* expression */) {

}

for(auto i = 0; i < 100; ++i) {

}
```

With for loops you can leave out one, or multiple of the parts of the loop such as:

```c++
auto i = // complex computation
for(; i < 100; ++i){

}
```

There is an actual difference between pre and post-increment (`++`). Applying `++` before a variable increments the variable and returns the new value. 
Applying `++` after the variable returns the old value and increments the variable. 
Thus, pre-increment can have better performance since it does not need to create a copy of the old value. 
For integers, the difference is negligible but for iterators or a user-defined type that implements `operator++()`, this may be important.

#### While Loop

```c++
while(/*condition*/) {

}

do {

} while(/*condition*/);
```

#### Switch

```c++
auto num = // ...

switch(num) {
    case 0:
    {
        // use {} to create a new scope for new variables
        auto res = /* ... */;
        break;
    }
    case 1:
        // ... 
        break;
    case 2:
    case 3:
        // this code executed when num is 2 or 3
        // ...
        break;
    case 4:
        // .. some code
    [[fallthrough]] // denote that we intend this case to fall through
    case 5:
        // code executed when num is 4 or 5
        break;
    default:
        // when num is everything else
}       


//Simpler Example:

switch(num) {
    case 0:
        //...
        break;
    case 1:
        // ...
        break;
    default:
        break;
}
```

Switches are pretty much just glorified `goto`s, and we'll talk about how they work under the hood later. 
The variable being switched upon must be an integral type such as a `char`, `int`, `short`, etc. Therefore, unlike other languages, strings don't work in switches. 
The code will jump to the case labelled with the value that is equal to the variable being switched upon. From there it will continue executing line after line until it reaches a `break`. 
This allows us to fallthrough from one case to another. Intended fallthrough should be avoided, but if necessary it should be labelled with the `[[fallthrough]]` attribute like shown above.

#### Infinite Loop

```c++

for(;;) {

}

while(true) {

}
```

If you leave out the guard of the `for` loop like shown above, a constant which evaluates to `true` is put in its place.

#### Break and Continue

```c++
for(/*...*/) {
    if(/*...*/) break; //exit loop
    else if(/* ... */) continue; //skip rest of code and go to next loop iteration
}

for(/*...*/) {
    for(/*...*/){
        if (/*...*/) goto dblBreak;
    }
}
dblBreak:
```

`goto` is not something you generally want to use (or even have to). 
Here's a funny story: the first time I needed an unconditional jump, I actually didn't know C++ had a `goto`, so instead I used inline assembly only to think how
pointless it was to write something so simple in assembly. And that's the day I discovered C++ did indeed have a `goto`. 
The only good usage I can think of off the top of my head is to break out of nested loops like so. We first create a label by typing a name followed by a colon, then we put that name after `goto` to jump to it.

### Comma Operator

The comma operator will execute the operations in order, from left to right but will only return the result of the last expression.

```C++
const auto y = 10, 20;
// y = 20;

const auto x = y++, ++y, y++;
// y = 23
// x = 22
```

### Ternary Operator

This operator looks like `<condition> ? <true_branch> : <false_branch>`.
If the condition is true, it will evaluate the expression on the left side of the colon,
otherwise it will evaluate the expression on the right side.

```C++
const auto age = 10;

const auto free_time = age < 16 ? 8 : 1;
// free_time is 8
```

#### Further Reading
[C++ Primer 5th Ed](https://github.com/yanshengjia/cpp-playground/blob/master/cpp-primer/resource/C%2B%2B%20Primer%20(5th%20Edition).pdf) 1.4 - 1.4.4 and 5.3 - 5.5.3