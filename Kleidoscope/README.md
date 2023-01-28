# The Kaleidoscope Language

This tutorial is illustrated with a toy language called “Kaleidoscope” (derived from “meaning beautiful, form, and view”). Kaleidoscope is a procedural language that allows you to define functions, use conditionals, math, etc. Over the course of the tutorial, we’ll extend Kaleidoscope to support the if/then/else construct, a for loop, user defined operators, JIT compilation with a simple command line interface, debug info, etc.

We want to keep things simple, so the only datatype in Kaleidoscope is a 64-bit floating point type (aka ‘double’ in C parlance). As such, all values are implicitly double precision and the language doesn’t require type declarations. This gives the language a very nice and simple syntax. For example, the following simple example computes Fibonacci numbers:

```python
# Compute the x'th fibonacci number.
def fib(x)
  if x < 3 then
    1
  else
    fib(x-1)+fib(x-2)

# This expression will compute the 40th number.
fib(40)
```

We also allow Kaleidoscope to call into standard library functions - the LLVM JIT makes this really easy. This means that you can use the ‘extern’ keyword to define a function before you use it (this is also useful for mutually recursive functions). For example:

```C++
extern sin(arg);
extern cos(arg);
extern atan2(arg1 arg2);

atan2(sin(.4), cos(42))
```

# The Lexer

When it comes to implementing a language, the first thing needed is the ability to process a text file and recognize what it says. The traditional way to do this is to use a “lexer” (aka ‘scanner’) to break the input up into “tokens”. Each token returned by the lexer includes a token code and potentially some metadata (e.g. the numeric value of a number). 

Now, we may refer to the "Lexer" part of the code.

# The Parser

The parser we will build uses a combination of *Recursive Descent Parsing* and *Operator-Precedence Parsing* to parse the Kaleidoscope language (the latter for binary expressions and the former for everything else). Before we get to parsing though, let’s talk about the output of the parser: the Abstract Syntax Tree.

## AST

The AST for a program captures its behavior in such a way that it is easy for later stages of the compiler (e.g. code generation) to interpret. We basically want one object for each construct in the language, and the AST should closely model the language. In Kaleidoscope, we have expressions, a prototype, and a function object. 

In Kaleidoscope, functions are typed with just a count of their arguments. Since all values are double precision floating point, the type of each argument doesn’t need to be stored anywhere. In a more aggressive and realistic language, the “ExprAST” class would probably have a type field.

Now, we may refer to the "AST" part of the code.

## Parser

Now that we have an AST to build, we need to define the parser code to build it. The idea here is that we want to parse something like “x+y” (which is returned as three tokens by the lexer) into an AST that could be generated with calls like this:

```C++
auto LHS = std::make_unique<VariableExprAST>("x");
auto RHS = std::make_unique<VariableExprAST>("y");
auto Result = std::make_unique<BinaryExprAST>('+', std::move(LHS), std::move(RHS));
```

The parser uses look-ahead to determine which sort of expression is being inspected, and then parses it with a function call.

Operator-Precedence Parsing uses the precedence of binary operators to guide recursion. The basic idea of operator precedence parsing is to break down an expression with potentially ambiguous binary operators into pieces. Consider, for example, the expression $a+b+(c+d)*e*f+g$. Operator precedence parsing considers this as a stream of primary expressions separated by binary operators. As such, it will first parse the leading primary expression $a$, then it will see the pairs $[+, b]$ $[+, (c+d)]$ $[*, e]$ $[*, f]$ and $[+, g]$. Note that because parentheses are primary expressions, the binary expression parser doesn’t need to worry about nested subexpressions like (c+d) at all.

Now, we may refer to the "Parser" part of the code.

# Code Generation

The codegen() method says to emit IR for that AST node along with all the things it depends on, and they all return an LLVM Value object. “Value” is the class used to represent a “Static Single Assignment (SSA) register” or “SSA value” in LLVM.

Note that instead of adding virtual methods to the ExprAST class hierarchy, it could also make sense to use a visitor pattern or some other way to model this.

Now, we may refer to the "Code Generation" part of the code.

# Optimizer

In the last section, the IRBuilder, does give us obvious optimizations when compiling simple code:

```bash
ready> def test(x) 1+2+x;
Read function definition:
define double @test(double %x) {
entry:
        %addtmp = fadd double 3.000000e+00, %x
        ret double %addtmp
}
```

With LLVM, you don’t need this support in the AST. Since all calls to build LLVM IR go through the LLVM IR builder, the builder itself checked to see if there was a constant folding opportunity when you call it. If so, it just does the constant fold and return the constant instead of creating an instruction.

Well, that was easy :). In practice, we recommend always using IRBuilder when generating code like this. It has no “syntactic overhead” for its use (you don’t have to uglify your compiler with constant checks everywhere) and it can dramatically reduce the amount of LLVM IR that is generated in some cases (particular for languages with a macro preprocessor or that use a lot of constants).

On the other hand, the IRBuilder is limited. We need more optimizations. Fortunately, LLVM provides a broad range of optimizations that you can use, in the form of “passes”.

## Passes

 LLVM allows a compiler implementor to make complete decisions about what optimizations to use, in which order, and in what situation.

 As a concrete example, LLVM supports both “whole module” passes, which look across as large of body of code as they can (often a whole file, but if run at link time, this can be a substantial portion of the whole program). It also supports and includes “per-function” passes which just operate on a single function at a time, without looking at other functions.

 For now, we will choose to run a few per-function optimizations as the user types the function in. If we wanted to make a “static Kaleidoscope compiler”, we would use exactly the code we have now, except that we would defer running the optimizer until the entire file has been parsed.

 In order to get per-function optimizations going, we need to set up a FunctionPassManager to hold and organize the LLVM optimizations that we want to run. Once we have that, we can add a set of optimizations to run. We’ll need a new FunctionPassManager for each module that we want to optimize, so we’ll write a function to create and initialize both the module and pass manager for us.

 Now, we may refer to the "Top-Level parsing and JIT Driver" part of the code.

 LLVM provides a wide variety of optimizations that can be used in certain circumstances. Some documentation about the various passes is available, but it isn’t very complete. Another good source of ideas can come from looking at the passes that Clang runs to get started. The “opt” tool allows you to experiment with passes from the command line, so you can see if they do anything.

 # JIT

 The basic idea that we want for Kaleidoscope is to have the user enter function bodies as they do now, but immediately evaluate the top-level expressions they type in. For example, if they type in “1 + 2;”, we should evaluate and print out 3. If they define a function, they should be able to call it from the command line.

In order to do this, we first prepare the environment to create code for the current native target and declare and initialize the JIT. This is done by calling some "InitializeNativeTarget" functions and adding a global variable TheJIT, and initializing it in main.

Now, we may refer to the "main" function of the code.