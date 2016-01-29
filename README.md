# The Smallest Compiler

Today we're going to implement a compiler.  It will compile a very small
language – integers.  That is, it will take a user program (a number), and
create an executable binary that prints the number.  There are no files in
this repository because the point of the lab is for you to see how this is
built up from scratch.  That way, you'll understand the infrastructure that
future assignments' support code will use.

## The Big Picture

The heart of each compiler we write will be an OCaml program that takes an
input program and generates assembly code.  That leaves open a few questions:

- How will the input program be handed to, and represented in, OCaml?
- How will the generated assembly code be run?

Our answer to the first question is going to be simple: we'll expect that all
programs are files containing a single integer, so there's little
“front-end” for the compiler to consider.  Most of this lab is about the
second question – how we take our generated assembly and meaningfully run it
while balancing (a) avoiding the feeling that there's too much magic going on,
and (b) getting bogged down in system-level details that don't enlighten us
about compilers.

## The Wrapper

(The idea here is directly taken from [Abdulaziz
Ghuloum](http://scheme2006.cs.uchicago.edu/11-ghuloum.pdf)).

Our model for the code we generate is that it will start from a C-style
function call.  This allows us to do a few things:

- We can use a C program as the wrapper around our code, which makes it
  somewhat more cross-platform than it would be otherwise
- We can defer some details to our C wrapper that we want to skip or defer
  until later

So, our wrapper will be a C program with a traditional main that calls a
function that we will define with our generated code:

```
#include <stdio.h>

extern int our_code_starts_here() asm("our_code_starts_here");

int main(int argc, char** argv) {
  int result = our_code_starts_here();
  printf("%d\n", result);
  return 0;
}
```

So right now, our compiled program had better return an integer, and our
wrapper will handle printing it out for us.  We can put this in a file called
`main.c`.  If we try to compile it, we get an error:

```
⤇ clang -o main main.c
Undefined symbols for architecture x86_64:
  "our_code_starts_here", referenced from:
      _main in main-1a486d.o
ld: symbol(s) not found for architecture x86_64
clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

That's because it's our job to define `our_code_starts_here`.  That's what
we'll do next.

## Hello, NASM

Our next goal is to:

- Write an assembly program that defines `our_code_starts_here`
- Link that program with `main.c` so we can run it

In order to write assembly, we need to pick a syntax and an instruction set.
We're going to generate 32-bit x86 assembly, and use the so-called Intel
syntax (there's also an AT&T syntax, for those curious), because [I like a
particular guide](http://www.cs.virginia.edu/~evans/cs216/guides/x86.html)
that uses the Intel syntax, and because it works with the particular assembler
we'll use.

Here's a very simple assembly program, matching the above constraints, that
will act like a C function of no arguments and return a constant number (`37`)
as the return value:

```
section .text
global our_code_starts_here
our_code_starts_here:
  mov eax, 37
  ret
```

The pieces mean, line by line:

- `section .text` – Here comes some code, in text form!
- `global our_code_starts_here` – This assembly code defines a
  globally-accessible symbol called `our_code_starts_here`.  This is what
  makes it so that when we generate an object file later, the linker will know
  what names come from where.
- `our_code_starts_here:` – Here's where the code for this symbol starts.  If
  other code jumps to `our_code_starts_here`, this is where it begins.
- `mov eax, 37` – Take the constant number 37 and put it in the register
  called `eax`.  This register is the one that compiled C programs expect to
  find return values in, so we should put our “answer” there.
- `ret` – Do mechanics related to managing the stack which we will talk about
  in much more detail later, then jump to wherever the caller of
  `our_code_starts_here` left off.



