
# `libscript` - A scripting library written in C++

[![Build Status](https://api.travis-ci.com/strandfield/libscript.svg?branch=master)](https://travis-ci.com/github/strandfield/libscript)
[![codecov](https://codecov.io/gh/strandfield/libscript/branch/master/graph/badge.svg?token=ehzgo7KnfP)](https://codecov.io/gh/strandfield/libscript)
[![Codacy Badge](https://app.codacy.com/project/badge/Grade/6e36741a37dd41879591324ac8093f19)](https://www.codacy.com/gh/strandfield/libscript/dashboard?utm_source=github.com&amp;utm_medium=referral&amp;utm_content=strandfield/libscript&amp;utm_campaign=Badge_Grade)

`libscript` is a scripting library written in C++. 
It aims at providing all the building blocks necessary to create 
statically typed languages that interface well with C++ : its goal
is not to be a full-featured language.

The library is licensed under the permissive MIT license and can therefore be 
used freely. 
However, it is still under development and is far from being stable enough 
for use in production.

The library follows the [Semantic Versioning](https://semver.org/).
The current version is `0.2.0`.
The API is subject to a lot a change.


## Using the library

No binaries are provided, you must compile the library by yourself.

### Building

**Requirements:**
- CMake
- C++14

**Cloning:**

```bash
git clone http://github.com/strandfield/libscript.git
```

**Build files:**

You can then use CMake to generate project files for your favorite build system 
with the following commands or using `cmake-gui`.

```bash
mkdir build && cd build
cmake ..
```

**Building on Linux:**

```bash
make
```

### CMake

Alternatively, you can add this repository as a subdirectory of a CMake project:

```cmake
add_subdirectory(libscript)
```

### Documentation

An incomplete, very basic documentation is available at [strandfield.github.io/libscript-docs](https://strandfield.github.io/libscript-docs/).

This documentation was generated from the source files using [dex](https://github.com/strandfield/dex/tree/develop).

## Example

### Hello World!

The classical "Hello World!" program:
```cpp
print("Hello World!");
```

And the C++ code required to run it (see [examples/helloworld.cpp](examples/helloworld.cpp)):
```cpp
#include "script/engine.h"
#include "script/function.h"
#include "script/functionbuilder.h"
#include "script/namespace.h"
#include "script/script.h"
#include "script/interpreter/executioncontext.h"

#include <iostream>

script::Value print_callback(script::FunctionCall* c)
{
  std::cout << c->arg(0).toString() << std::endl;
  return script::Value::Void;
}

int main()
{
  using namespace script;

  Engine e;
  e.setup();

  FunctionBuilder::Fun(e.rootNamespace(), "print")
    .setCallback(print_callback)
    .params(Type::cref(Type::String))
    .create();

  const char* source = R"(
     print("Hello World!");
  )";

  Script s = e.newScript(SourceFile::fromString(source));
  
  if (s.compile())
    s.run();
}
```

### Gonk!

For a more realistic example, see the [gonk](https://github.com/strandfield/gonk) 
project.

`gonk` is a scripting language built with `libscript`.
It has a module system based on plugins (dynamic libraries).

`gonk` also provides an interactive mode and a basic GUI debugger written 
with `Qt`.

## Features

The language is statically typed and has a C++ like syntax.
The idea is to have a good type system that can catch a many errors 
before ever running the code.

The language provides 5 fundamental types:
  - `bool` : `true` or `false`;
  - `char` : ASCII characters;
  - `int` : integers;
  - `float` : simple precision floating point numbers like `3.14f`;
  - `double` : double precision floating point numbers (e.g. `3.14`, `1.2345e6`).

It also provides an `Array` type and a `String` type. 
```cpp
Array<int> a = [1, 2, 3, 4, 5];
String str = "Hello " + "World !";
```
There is no "pointer" semantic so, unlike in C++, strings can be concatenated using `+`. 

The language supports both object-oriented programming and generic programming 
with classes and templates. 
Functional programming can be achieved partially with lambdas. 

```cpp
class Point
{
public:
  int x; int y;
  Point() = default;
  ~Point() = default;
  /* ... */
};

template<typename T>
const & T max(const T & a, const T & b)
{
  return a > b ? a : b;
}

auto func = [](int a){ return a + 2; };
```

Operator overloading, user-defined conversions and literal operators are also supported.

Basic support for initializer lists is available.

```cpp
int sum(InitializerList<int> list)
{
  int s = 0;
  for(auto it = list.begin(); it != list.end(); ++it)
  {
    s += it.get();
  }
  return s;
}
int n = sum({1, 2, 3, 4});
```

Unlike C++, libscript does not rely on a preprocessor and has no `#include` mechanism.
Instead, a very basic module system allows the importation of modules written in C++ 
or user-defined scripts.

```cpp
// file: a
int foo() { return 42; }

// file: b
import a;
int n = foo();
```

## How it works

The main task of `libscript` is to execute script files. 
The processing of scripts is split into three stages:
  - lexical analysis
  - syntactical analysis
  - semantic analysis

The first step, namely lexical analysis, consists of extracting 
tokens (sometimes referred to as lexeme) from the input file. 
This stage is performed by a lexer (see [lexer.cpp](src/parser/lexer.cpp)). 

Considering the following input, 
```cpp
int a = 5;
```
the lexer will output the following tokens:
  - `int` : identifier, keyword
  - `a` : user-defined name
  - `=` : operator, equal-sign
  - `5` : literal, integer
  - `;` : punctuator, semicolon

The second stage consists of creating a syntactical tree out of the 
created tokens. This tree is often called an AST, which stands for 
*abstract syntax tree*.
This is done by the `Parser` class in [parser.cpp](src/parser/parser.cpp).

A simplified representation of the parser's output for the previous 
example is given hereafter:
```js
VariableDeclaration(
  type : `int`,
  name : `a`,
  initialization : Assignment(
    value : `5`
  )
)
```

The syntactical analysis is much more complicated than the lexical 
analysis. 
For example, some constructions may be difficult to distinguish because 
they are similar.
Consider the following,
```cpp
A B(C, D);
A B(C, D) { }
```
The first one is interpreted by our parser as a variable declaration 
having type `A` and name `B`, and constructed using two arguments: 
`C` and `D`.
The second one is interpreted as a function definition returning `A` 
and taking as inputs two parameters of type `C` and `D`.
As you can see, we need to wait until the semicolon or the opening 
brace to make the decision.

A program need not be correct to pass this stage; the only requirement 
is that the syntax matches the language's constructs.

The last stage consists of converting the AST into an *executable* tree 
that is easy to interpret.
This is done by the `Compiler` class.
This stage involves two important processes called *name lookup* and 
*overload resolution*. The first one consists of searching for what a name 
refers to (is `foo` the name of a variable or the name of a set of functions ?), 
while the second one consists of selecting the *best* function from a set 
of candidate functions. The definition of *best* is not simple and will not 
be discussed here.

If you want to learn more about overload resolution and name lookup, you can 
read how they work in C++ on [cppreference.com](http://en.cppreference.com/w/):
[name lookup](http://en.cppreference.com/w/cpp/language/lookup); 
[overload resolution](http://en.cppreference.com/w/cpp/language/overload_resolution) .

# Goals & next steps

The goal of this project was to get a better understanding of how C++ works 
by implementing some of its features as a (C++) library.

Some feature ideas are listed in the [project's backlog](https://github.com/strandfield/libscript/projects/1), 
including implementing iterators and a simple exception system; but there is 
no deadline or real schedule on when this will be done.
