\subsection{Concepts}\index{C++20!concepts}\index{concepts}

Concepts are probably my favorite (so far). I showed you an example before but now I'm going to take some more time to explain them. Concepts are essentially "meta-types". They provide a type system to templates. Here's the motivation: not every template function can do something meaningful for every type. Instead of producing strange error messages when a type that doesn't provide the expected interface is used (In GCC I once got one over a couple hundred lines, which is rookie numbers) the compiler will fail a little more gracefully. We "solved" this already with SFINAE, but concepts provides greater language support.

Let's start with the \inlinecode{requires} clause, aka constraints\index{constraints}. The requires keyword can be added after the function declaration or the template declaration, and imposes some constraint on the types that the template parameter can bind to. It takes a constant boolean expression that when false, prevents that type from binding.

\begin{lstlisting}[language=C++]
template<typename T>
    requires std::is_arithmetic_v<T>
T add5(T t) {
    return t + 5;
}

auto r = add5(20);
auto bad = add5("Hello"); //error!
\end{lstlisting}
As you can see, this dovetails nicely with pre C++20 type traits\index{type traits}. 

This is all well and good, but for complicated type checks, this is quite annoying. That's why we can define our own concepts. A concept is a special type of constant boolean expression (in the TS, you'd put bool before concept in the definition). In fact, concepts \textit{do} evaluate to booleans, so you can use them anywhere you can use a bool. One way to define them is as such:

\begin{lstlisting}[language=C++]
template<typename T>
concept MyConcept = std::is_arithmetic_v<T> || 
                    std::is_pointer_v<T>    ||
                    std::is_enum_v<T>;
                    
template<MyConcept MC>
void do(MC m);

template<typename T>
    requires MyConcept<T>
void do2(T m);

template<typename T>
void do3(T m) requires MyConcept<T>;

static_assert(MyConcept<int>); 
// use it as a bool

template<typename T>
concept MyConcept2 = MyConcept<T> &&
    std::is_signed_v<T>;
// Define a concept in terms of another
\end{lstlisting}
A concept can be used as the "type's type" in a template argument, or can be used in a constraint. In this example the concept is set equal to some constant expression and use it by having it as a constraint via \inlinecode{requires} and in the template argument.

Let's see a little more complex examples.
\begin{lstlisting}[language=C++]
template<typename T>
concept SingletonDeadReferencePolicy = requires(T a) {
	T::onDeadReference();
};
// type must have a static function 
// onDeadReference()

template<template <typename> typename T, typename R>
concept SingletonCreatePolicy = requires(T<R> a) {
	{T<R>::create()} -> std::same_as<R*>;
	{T<R>::free(std::declval<R*>())} noexcept;
};
// type must have a static function create() the returns an R*
// must also have a static, noexcept free() function that
// takes an R*


template<typename R, typename T>
concept StaticCastableTo = requires(T a) {
	static_cast<R>(a);
};
// checks if T can be cast to R
// notice the use of a, which is an "instantiation" of the type T
\end{lstlisting}
Concepts can have constraint blocks as shown. In the constraint will be a series of expressions that are unevaluated. Type T satisfies the concept if all expressions are valid. We can also check the return type by wrapping an expression in \{\}, followed by $\Rightarrow$. What goes to the right of the arrow is not any arbitrary boolean expression such as \inlinecode{std::is_same_v<T>}, but it must be another concept. Standard concepts such as \inlinecode{std::same_as} are defined in the concepts header.

The requires blocks can take a "parameter", this gives you access to check for non-static properites. It basically is the same as what we'd use std::declval for in pre C++20.

Concepts can also be used to check for member type aliases, and we can nest other concepts as well.

\begin{lstlisting}[language=C++]
template<typename T>
concept TList = requires(T list) {
	typename T::Value;
	typename T::Next;
};
// Type must have a member type alias called Value
// and Next

template<typename T>
concept MyConcept = requires(T a) {
	{ typename T::Value } -> std::same_as<int>; 
	//requires to have a typedef/using for int
	{a.member} -> std::derived_from<Base>; 
	//requires member derived from Base
	{a.otherMember} -> std::same_as<bool&>;
	// otherMember is type bool
	{a.function()} -> std::is_convertable_to<int>; 
	//function that returns something that can be an int
	
	requires std::derived_from<TBase>;
	// type type must derive from TBase
}
\end{lstlisting}

Concepts are excellent for PBD and templates. Prefer concepts over SFINAE.