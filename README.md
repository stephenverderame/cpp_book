# C++ Reference Book

This book was made to provide be a learning resource for new C++ systems developers who have decent programming experience in another language. 
It also can serve as a reference for experienced devs as well.

This markdown book is actually version 2 of a [latex document I wrote up last year](https://www.dropbox.com/s/ph0aw02phxleo3w/Cpp_as_fast_as_possible.pdf?dl=0). This version is far superior,
however it does lack some things such as an index due to being written in markdown.

## Usage

Since I wanted this to be able to serve as a reference, it does contain information on quite a lot of specific language details that someone new to C++ probably doesn't need to know.
Reading this front-to-back might be a bit much, but there are many things that don't have their own chapter, and I simply discuss them as they appear in examples. Therefore,
you might find it useful to read this front-to-back and just not worry about memorizing nitty gritty details.

I have provided possible exercises, as practice is definitely what makes this information stick. I plan to provide possible solutions to these, but as of now I haven't gotten around to it.

The book also outlines a possible learning project in the last chapter, and starter code is provided.

## Building

This book is generated from `mdbook` which can be installed from Rust cargo. You can find the source [here](https://github.com/rust-lang/mdBook). 
To build it, simply run the command `mdbook build` in the top-level directory with the `book.toml` file.

For editing the markdown, I found IntelliJ to be really helpful thanks to the addon [Grazie](https://blog.jetbrains.com/idea/2019/11/meet-grazie-the-ultimate-spelling-grammar-and-style-checker-for-intellij-idea/) 
which provides grammar and spell checking. VS Code also has a  [code spell checker](https://marketplace.visualstudio.com/items?itemName=streetsidesoftware.code-spell-checker) extension, but I didn't find it as
helpful.