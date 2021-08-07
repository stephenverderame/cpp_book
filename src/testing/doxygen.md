# Doxygen

Doxygen is a popular tool to generate documentation from the comments we write in code. It can generate HTML, XML, LaTeX, and more. Doxygen recognizes a lot of commenting styles but probably the most popular are three slashes `///` and Javadocs style `/** */`. If using Javadocs, we'll probably want to set `JAVADOCS_AUTOBRIEF` to true in the configuration file so the first line (until the first period) is recognized as a brief description. Otherwise, a brief description will require an explicit `@brief` tag.

Documentation comments typically go right before the thing they are documenting. We can put them right after however, if the first character of the comment is `<`.

```C++
/// An IO stream to a hardware port
class Port {
protected:
    std::optional<std::string> lastError; 
    ///< The most recent error since errstr() was called.
public:
    virtual ~Port() = default;

    /**
    * Writes all of data to the port
    * Provides the basic guarantee
    * @throw IOException on failure
    */
    virtual void writeAll(const std::vector<std::byte> & data) = 0;

    /**
    * Gets the last error and resets the error string
    * @return the most recent error since the last call to
    *   errstr or empty optional if there has been no error
    */
    std::optional<std::string> errstr() noexcept {
        const auto err = lastError;
        lastError.reset();
        return err;
    }

    /**
    * Reads at least the specified amount of bytes from the port
    * Call will block until specified amount of bytes are read
    * @param bytes the minimum amount of bytes to read
    * @return data read or empty vector if nothing read. Size will be at least `bytes`
    * @throw IOException on failure - Basic guarantee
    */
    virtual std::vector<std::byte> read(size_t bytes = 0) = 0;
};

```

I tend to use `///` for short or single line comments and Javadocs for longer ones. I also tend to omit things that I feel are unnecessary or repetitive such as an `@param` indicator for `data` in `writeAll()`. Based on this logic, `@param bytes` could probably be removed from the spec of `read()` however I personally would rather slightly verbose comments than slightly ambiguous.

Since we're looking at this code, I probably wouldn't have a `lastError` member as we can get this information to the caller in our `IOException`. Plus, the comment for `lastError` is slightly repetitive but this was really more for examples than anything.

Doxygen supports markdown in their comments. They also have a large range of special commands denoted with either a backslash or at sign and then the command name.

One such category of these commands are structural commands which allow you to document something separate from its definition. The syntax would be to put the structural command on the first line of the file followed by the name of the thing you are commenting. Structural commands include:
* `\file` - comment for file
* `\def` - document a `#define`
* `\var` - variable
* `\fn` - function
* `\struct`
* `\namespace`
* `\union`
* `\enum`


```C++
/** \file source.cpp
* Doxygen testing
* Detailed Desc
*/

/// \def MAX(a, b)
/// max value of `a` and `b`

/// \namespace Utils
/// Contains utility functions



#define MAX(a, b) ((a) > (b) ? (a) : (b))

namespace Utils {

}
```

Doxygen allows you to group functions under the same comment with member grouping. A group is opened with `@{` and closed with `@}`. This allows you to use one comment for similar functions such as overloads. Groups can also be named using `@name` (or `\name`) before the opening bracket. The following example is taken from Doxygen documentation.

```C++
/** A class. Details */
class Memgrp_Test
{
  public:
    ///@{
    /** Same documentation for both members. Details */
    void func1InGroup1();
    void func2InGroup1();
    ///@}
 
    /** Function without group. Details. */
    void ungroupedFunction();
    void func1InGroup2();
  protected:
    void func2InGroup2();
};

/** @name Group2
 *  Description of group 2. 
 */
///@{
/** Function 2 in group 2. Details. */
void Memgrp_Test::func2InGroup2() {}
/** Function 1 in group 2. Details. */
void Memgrp_Test::func1InGroup2() {}
///@}
 
```

Doxygen also supports module grouping which puts group members onto separate pages. To do this, first you must create a group by using `\defgroup` followed by the group label and then a description if desired. Next, we can add definitions to that group by putting `\ingroup <group label>` in the documentation comment of that member. To avoid repeating this command, you can also put `\addtogroup <label>` before the opening member group brace (`@{`) to add everything within that member group to the module group. Finally, you can embed links to another group using the `\ref <label name> ["<link text>"]` command and specifying an optional string to use as the hyperlink in the comment.

```C++
/** @defgroup Group1
*  Basic Math functions
* @{
*/

int add(int, int);
int sub(int, int);
///@}

/// \ingroup Group1
int div(int, int);

/// \addtogroup Group1
/// @{
int mul(int, int);
int pow(int, int);
/// @}

/// \ref Group1 "Click Me"
void foo();
```

You can use Latex formulas in comments by enclosing them within `\f[` and `\f]`. This only works for HTML, Latex, and RTF output.

## Linking

Doxygen automatically links to other classes and files when we include them in the comment. All words with a dot that isn't the last character are considered to be file names and if such a file was input to the generator, a link to that file is automatically generated. Doxygen will automatically link to classes if the class names contains a least one non-lower-case character (uppercase, underscores, etc.). If a class contains only lowercase characters, you can still link to it with `\ref`.

Functions are automatically linked when it encounters patterns including the following:
* `<function name>()` or `<function name>(<args>)`
* `::<function name>`
* `<class name>::<function name>[([<args>])] [<modifiers>]`
    * Examples:
        * `MyClass::myFunc`
        * `MyClass::doFoo(int, int) const`
        * `MyClass::doIt(int)`

Arguments and modifiers are required if there are overloads of the same function. Just like all `@` can be replaces with `\` and vice versa in commands, so too can we replace `::` with `#`.

Doxygen also supports the `a` HTML tag for links as well (`<a href="https://en.wikipedia.org/wiki/Main_Page">Click Me</a>`). In comments, we can use `\sa` or `\see` to list any references, although this section isn't necessary. This section creates a separate paragraph in the output whereas just using automatic link generation does not.

## Other useful commands:

* `\todo {desc of todo}`
* `\warning {desc of warning}`
* `\bug {desc of bug}`
* `\emoji <emoji name>`
* `\image <format> <file> [<caption>] [<dimension> = <size>]`
    * `format` indicates the output format in which the image should be embedded: `html`, `rtf`, `latex`, or `docbook`.
    * `dimension` is used to specify either the width or height of the image
    * `\image html myImg.jpg`
* `\internal` and `\endinternal`
    * Comments for internal use only
* `\page <name> <page tile>`
    * Indicates the comment belongs on a separate page and is separate from documentation of a method or class
    * In a comment block that is a page comment, we can create sections with:
        * `\section <name> <section title>`
        * `\subsection <name> <subsec title>`
        * `\subsubsection <name> <subsubsec title>`
        * `\paragraph <name>`
        * `\subpage <name>` - indicates that page `name` is a child of the current page comment block
        * `\tableofcontents` - creates a TOC for the current page and all sections
    ```
    From Doxygen documentation:


    /*! \page page1 A documentation page
    \tableofcontents
    Leading text.
    \section sec An example section
    This page contains the subsections \ref subsection1 and \ref subsection2.
    For more info see page \ref page2.
    \subsection subsection1 The first subsection
    Text.
    \subsection subsection2 The second subsection
    More text.
    */
    
    /*! \page page2 Another page
    Even more info.
    */
    ```
* `\mainpage <name> <title>`
    * Same idea as `\page` except this page is the main page. For example, it would become the `index.html` page in html output.
* `\anchor <name>`
    * Creates a point that can be referenced with `\ref name`
* `\pre {desc}` - starts a paragraph for preconditions
* `\post {desc}` - starts a paragraph for postconditions
* `\test {desc}` - starts a paragraph for test cases or examples
* `\invariant {desc}` - starts a paragraph for invariants
* `\implements <name>` - indicates a subtyping relationship. Not necessary when using OOP inheritance

## Config

With the code documented, we can generate the config file with the command `doxygen -g <config-file-name>`. This command can be done in the root directory of your project. This will generate a template config file which, for the most part is pretty good. A few things I do like to change is to set `SEARCHING` to `YES` and `SERVER_BASED_SEARCH` to `NO`. This will allow client-side searching in the generated output. For HTML, this is a local javascript search engine. I also set `JAVADOCS_AUTOBRIEF` to `YES` so that the first line of the documentation is the brief without having to specify `\breif`. You probably also want to set `RECURSIVE` to `YES` so that it will recursively go though the directory looking for files that match supported file patterns. These patterns can be modified by changing the `FILE_PATTERNS` setting and the `EXCLUDE_PATTERNS` setting. You can also add files to the `EXCLUDE` setting to exclude certain files or directories.

We can control where and how doxygen will generate the output with the `OUTPUT_DIRECTORY`, `HTML_OUTPUT`, `LATEX_OUTPUT`, `RTF_OUTPUT`, `XML_OUTPUT`, `DOCBOOK_OUTPUT`, and `MAN_OUTPUT` tags. By default the output directory is the same directory as the doxyfile (doxygen config file).

Finally, to generate our documentation, run `doxygen <config file name>`.