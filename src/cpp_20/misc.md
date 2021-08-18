\subsection{Rocketship Operator}\index{rocketship operator}
\index{C++20!rocketship operator}

This is actually called the 3-way-comparison operator\index{3-way-comparison}, but it's colloquially known as the Rocketship Operator and I basically refuse to call it anything else.\footnote{I will similarly always use the term "dirty bit" and never "update flag" or "update bit" or "needsRefresh"}. This is very similar to Java's comareTo() method.

If you've ever implemented a comparable type, you may know the pain of manually defining all \inlinecode{<=, ==, !=, >= <, >} and then defining them again so that they are commutative. If so, then the three-way comparison \inlinecode{( <=> )} is what you need! Given the following syntax:\\
\inlinecode{a <=> b}\\
It returns:
\begin{itemize}
    \item $< 0$ if $a < b$
    \item $ = 0$ if $a == b$
    \item $ > 0$ if $a > b$
\end{itemize}
This isn't exactly true, what it actually returns is an object of type std::strong\_ordering or std::partial\_ordering, which are convertable to integers. \index{std::strong\_ordering}\index{std::partial\_ordering} std::strong\_ordering has 3 members, std::strong\_ordering::equal, less, and greater. std::partial\_ordering has 4: equivalent, less, greater and unordered.

The nice thing about our little rocketship here, is that we don't have to explicitly use it to take advantage of it. Consider:

\begin{lstlisting}[language=C++]
#include <compare>
struct Person {
    std::string name;
    int age;
    
    Person(std::string n, int a) 
        : name(n), age(a) {}
    
    auto operator<=>(const Person& other) {
        return age <=> other.age;
    }
    
    auto operator<=>(int age) {
        return this->age <=> age;
    }
}

Person p {"PersonA", 20};
Person p2 {"PersonB", 18};
Person p3 {"ABB", 18};

if(p2 < p) {
    //this executes
}
if(100 >= p) {
    //this executes
}
auto val = p2 == p3; //true
\end{lstlisting}
Traditionally, if we defined \inlinecode{>=} for an int and our type, we'd have to define two versions. One where the integer is the left hand parameter and the other where the integer is the right. With the spaceship operator, the compiler handles that for us. Indeed what's actually going on is the compiler is generating all the comparison functions for us. 

There is a slight problem. For some types, such as strings, some comparison operations are much faster than others, namely == and !=. The good news is that's perfectly fine because the compiler will implement == independently of the spaceship operator and handle that for you. If you had your own type with your own equality comparison that wasn't simply delegating the comparison to a member, you can simply define your own operator==(). Any manually defined comparison operators will take precedence over the rocketship operator.