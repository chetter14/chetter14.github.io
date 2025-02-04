---
layout: post
title: Develop My Programming Language. Part 1.
---

Here I am, after a few months of silence. Now I want to share my experience of making *some sort of* a **compiler**. The task is part of my studying at Master's.

So, basically I should write a program in *C*. The program takes an *input of one or more files that contain a code in my language* (well, the syntax it should follow is given by a teacher) and produce *an output - an executable file*.

The first thing to do is to build an **Abstract Syntax Tree (AST)**. For this purpose I use [ANTLR3](https://www.antlr3.org/). It's such a tool that allows you to write *syntax rules for your language*. I've written simple rules regarding: function signatures and calls, parameters, various expressions, types, and etc. All of it you can see [here](https://github.com/chetter14/my-language/blob/master/common/MyLanguage.g).

Using `antlr-3.5.3-complete.jar` on `MyLanguage.g` file I get lexer and parser sources and headers that are going to be utilized in my program sources.

Then `antlr-runtime-c` [library](https://github.com/antlr/antlr3/tree/master/runtime/C) was required for successful build. I've taken it by copy-pasting for the lack of time and experience of working with the third-party open-source libraries. Not sure whether it matters or not, I have no experience with software licenses as well.

So, to my code. I wrote a simple `main()` function that reads an input file and produces an AST structure. To present the result AST the [Graphviz](https://graphviz.org/) library is used. And thanks to `antlr-runtime-c` lib there is a function `makeDot()` that produces a file of `DOT` format that is readable by **Graphviz** library:
```
pANTLR3_STRING dotString = parser->adaptor->makeDot(parser->adaptor, tree);

fprintf(dotFile, "%s", (char*)dotString->chars);
fclose(dotFile);
```

There is a `.dot` file shows up after execution of the program. By the command `dot -Tpng output.dot -o output.png` it is being converted into `.png` format so that you can easily see the AST.

Also I made some modifications to the AST that is taken from *ANTLR* library functions, namely rotation of function calls and array accesses. `ANTLRv3` (the 3rd version) does not allow left-recursive expressions, that's why there is *suffixes* in syntax rules and not *prefixes*. This is requirement from teacher that function calls and array accesses should be left-recursive, and I did it.
