subsection{Modules}\index{module}

Modules\index{C++20!module} help with some of the problems of includes in C++. They are compiled independently of the transnational units\index{translation unit} that import them and unlike headers, are saved in a binary format. Thus once they're compiled, they are quick to load and remain unchanged and independent from the code that depends on them. They are similar to Java's .class files but with greater encapsulation. You may define a module in separate interface and implementation files, or all in one file. In modules, code is not available outside the module unless its explictly exported. Furthermore, macros and includes used in one module does not affect the code that imports the module. Here's an example:
\begin{lstlisting}[language=C++]
//begin interface file MyModule.ixx
//--------------------------------------
export module MyModule; 
//declare module interface
export import std.core;
//reexport std.core and import it
// (not something you need to do but I just wanted
// to show re-exporting)

namespace ModuleNS {
    int internalFunc();
    
    
    export int getInt(); 
    //give a function external linkage
}
//begin implementation file MyModule.cppm
//---------------------------------------
module MyModule; //module implementation, no export

int ModuleNS::internalFunc() {
    return //...
}
// no export keyword
int ModuleNS::getInt() {
    return ModuleNS::internalFunc();
}
//begin Main.cpp
//----------------------------------------
import MyModule;

int i = ModuleNS::getInt();
int bad = ModuleNS::internalFunc(); //error
\end{lstlisting}
For the interface file, we must explicitly export the things we want to make available to outside code.\index{export}An implementation file does not contain the keyword export, but it can import other modules. We can also define a module all in one file as well.

\begin{lstlisting}[language=C++]
export module Module2;


import std.core; //module version of standard library
#include <some_file.h> // only affects this module
// can include normal headers as well

import <iostream> //compatability with standard lib headers
#define INTERNAL_DEFINE_ONLY 1

export namespace EntireExportedNS {
    void exportedFoo() {
        std::cout << "Foo" << std::endl;
    }
    
    std::string exportedBar() {
        return "Bar";
    }
    
    template<typename T, typename U>
    auto sum(T a, U b) {
        return a + b;
    }
}
//template functions can be defined in modules
\end{lstlisting}
When we export a namespace like above, all the names in the namespace are exported. I'm not sure if this is a quirk but I have found that exporting a template directly does not seem to work, but it can be done if you put it in an exported namespace.


You may be curious how templates can be defined in modules if they are precompiled. Well that's because they aren't' exactly precompiled. Instead they're stored as abstract syntax trees.

Modules also have \textit{partitions}\index{module!partition}\index{C++20!module!partition} that allow an implementation to be split up into multiple files. Last time I checked, this is currently unsupported at least in MSVC. I also believe that modules are still in early stages for other compilers and is the most fleshed out in MSVC. 

But here's the partition syntax. A partition is specified following a colon and can be used exactly like a module.
\begin{lstlisting}[language=C++]
// Begin file hello.cpp
// ------------------------------
export module Hello:inter; 
// partition :inter of module Hello
#include <string_view>
// interface partition of Hello
export void SayHello
  (std::string_view const &name);

// Begin file helloImpl.cpp
// -------------------------------
module;
#include <iostream>
// implementation partition of Hello
module Hello:impl; 
import :inter; // import the interface partition
import <string_view>; 

using namespace std;
void SayHello (string_view const &name)
// matches the interface partitions’s exported
// declaration
{
  cout << "Hello " << name << "!\n";
}

// Begin file hello.ixx
// --------------------------------------
export module Hello;
// reexport the interface partition
export import :inter; 
import :impl; // import the implementation partition
// export the string header-unit
export import <string_view>;  
\end{lstlisting}
\footnote{\href{https://accu.org/journals/overload/28/159/sidwell/}{Source for code sample}}
To use in MSVC, module interfaces must have the .ixx file extension and module implementations must have the .cppm extension. You must also download the experimental module extensions to the C++ build tools and enable it with the compiler flag /experimental:module. Modules are currently unsupported in cmake with the ninja build system.
Here's a narrow case utility re-implemented with concepts and modules:
\begin{lstlisting}[language=C++]
export module Cast;
import std.core;

// module linkage definitions
// not visible to users
// integral to integral (or enum)
template<typename R, typename T>
requires 
    (std::is_arithmetic_v<R> && 
    (std::is_arithmetic_v<T> || std::is_enum_v<T>))
bool isSignMismatch(T val) {
	return val < static_cast<T>(0) && 
	    !std::is_signed_v<R>;
}
//integral to enum overload
template<typename R, typename T>
requires 
    (std::is_enum_v<R> && 
    (std::is_arithmetic_v<T> || std::is_enum_v<T>))
bool isSignMismatch(T val) {
	return (val < static_cast<T>(0) && 
	!std::is_signed_v<std::underlying_type_t<R>>);
}
//non integral types
template<typename R, typename T>
bool isSignMismatch(T val) {
	return false;
}
// these constraints can be cleaned up as concepts

// exporting the public interface
export namespace SUtil {
	template<typename R, typename T>
	concept StaticCastableTo = requires(T a) {
		static_cast<R>(a);
		// defines this concept as being able to
		// cast from the type in question to R
	};

	/**
	 * Safely casts <T> to <R> if value can fit inside <R>
	 */
	template<typename R, StaticCastableTo<R> T>
	    requires StaticCastableTo<T, R>
	R narrow_cast(T value) {
		if (static_cast<T>(static_cast<R>(value)) != value)
			throw std::bad_cast();
		return static_cast<R>(value);
	}

	/**
	 * Safely casts <T> to <R> if value can fit inside <R>
	 * Fails if value is < 0 and R is unsigned
	 */
	template<typename R, StaticCastableTo<R> T>
	    requires StaticCastableTo<T, R>
	R strict_narrow_cast(T value) {
		if (isSignMismatch<R>(value)) throw std::bad_cast();
		return narrow_cast<R>(value);
	}
}

// not part of narrow cast --------------
export class Person {
private:
    int age;
public:
    int getAge() {
        return age;
    }
}
\end{lstlisting}
While templates can be in modules, they only seem to be able to go in module interface files, for the same reasoning as having to put them in header files. I didn't try it, but I'm sure you can manually instantiate a finite number of templates the same way you would if you put a template definition in a source file.