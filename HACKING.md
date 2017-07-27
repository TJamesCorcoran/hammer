# Grammars: compiled vs uncompiled

Grammars represent things like "this character set", "this constant
string", "either this character set or that constant string", etc.

Simple grammars are composed into graphs by the coder, to give rise to
more complex grammars.

This method is easy to use, but it does not create an internal
representation that is performant.

Thus there is [ sometimes ] a desire to compile or "desugar" these
initial graphs of grammar objects into more compact / quicker objects.

The simplest case of using a parser without compiling it is this:

```
  const HParser *p = h_end_p();  // recognize the end of input
  HParseResult *res = h_parse(p, (uint8_t *)"a", 1);
```

With compiling it looks like this:

```
  const HParser *p = h_end_p(); 
  h_compile(p, PB_PACKRAT, (void *) NULL));
  HParseResult *res = h_parse(p, (uint8_t *)"a", 1);
```

Note that the compile step allows one to specify a different grammar than the default:

```
  const HParser *p = h_end_p(); 
  h_compile(p, PB_LALR, (void *) NULL));  // <----- LALR grammar specified
  HParseResult *res = h_parse(p, (uint8_t *)"a", 1);
```

More on grammars below.


# Backends


In computer science there is a hierarchy of classes of languages.
Each class n+1 contains all the languages in the class n, plus
additional languages.  This is called the [Chomsky
hierarcy](https://en.wikipedia.org/wiki/Chomsky_hierarchy):

(see also [a chart](https://en.wikipedia.org/wiki/Template:Formal_languages_and_grammars) )

* type-0:
  * [recursively enumerable](https://en.wikipedia.org/wiki/Recursively_enumerable_language)
  * ...
* type-1: context sensitive
  * context sensitive
  * indexed
  * ...
* type-2: context-free
  * context free
  * deterministic context free
  * ...
* type-3: regular
  * regular
  * ...

There is also this, which does not fit neatly in:

* PEG       - Parsing Expression Grammar 

Hammer supports multiple backends to allow coders to made tradeoffs in
the parsers they build between simplicity / speed and the power to
parse more complicated languages. These are enumerated in hammer.h in the enum `HParserBackend`,
and implemented in `/backends`:

*  PB_PACKRAT	- parses PEGs ( this is the default backend )
*  PB_REGULAR	- parses regular expressions (which are an example of type-3 regular languages)
*  PB_LALR		- parses DCFL languages (Look Ahead Left-to-Right parser)
*  PB_LLk		- parses a subset of CFL (Left Look Ahead with k tokens of lookahead)
*  PB_GLR		- parses all CLF (Generalized Left-to-Right)

A backend is specified when a parser is compiled.  If the parser is
not compiled, or if the parser is compiled with PB_PACKRAT, the
default grammar, PB_PACKRAT, is used.

Each backend is implemented by a tuple of functions ( compile(),
parse(), free(), parse_start(), etc.)

A given backend's tuple of functions is stored (via reference) in a
`HParserBackendVTable` - a simple struct of function pointers.

There is a global array of all the backend `HParserBackendVTable`
called - appropriately - `backends`.

The way that compiling works is that there is a global array of
vtables (each a struct of function pointers), one vtable per backend,
and the appropriate backend compilation function is invoked.

The vtable for a backend x may be found by `backends[x]`.

# Compilation / desugaring


## Theory

Desugaring is also called "sum-of-products" (or, rather, "converting into sum-of-products").

Why?
q
Imagine that I tell you about a mini language that I've created where
all valid sentences are defined by this pseudo code:

				 ( seq (choice A  B) (choice C  D))

if you go through and enumerate all the possible sentences, you'll see
that there are four of them:

				 ( seq (choice A  B) (choice C  D)) = (choice (seq A C) (seq A D) (seq B C) (seq B D))

...which looks quite similar, in form, to:

		 		   (a + b) * (c + d) = ac + ad + bc + bd

If you allow the analogy of "choice is like addition and
multiplication is like sequencing" (an arbitrary analogy chosen
because it makes things look nice), then this is like algebra's
distributive rule.

...and, more interestingly, it's a quicker to see if some input
matches

	(choice (seq A C) (seq A D) (seq B C) (seq B D))

than to see if it matches

	( seq (choice A  B) (choice C  D))

XXX more theory goes here

## Practice

All backend vtable contains a function `compile()`. Some are no-ops
(e.g. `h_packrat_compile()` ), but those that do anything
(`h_lalr_compile()`, `h_llk_compile()`, etc.) all work similarly:

1. builds a HCFGrammar object (an internal representation not exposed
   to the coder)

1. create a desugared representation of the parser data structure and
   store it in the parser (in the 'desugared' field)

1. generate a parse table (implementation specific to each backend) and
   store it in the parser (in the 'backend' field)

1. free the HCFGrammar object

Once the complilation step is done, the `HParser` "looks" the same to
the coder (it's still an opaque-ish struct), but it has been
instrumented w two pointers.



# Parsing


The function `h_parse()` when invoked on a HParser, finds the vtable
for the backend (using `backends[x]`), and then finds the function
pointer to the backend-specific parse function (e.g. `h_lalr_parse()`,
`h_llk_parse()`, etc.) and invokes it.


# Memory model

Parsers need to allocate and free memory.  Hammer could theoretically
use straight `malloc()` and `free()`, but a layer of abstraction is
nice because it lets the coder change the implementation.

`HAllocator` is a vtable that contains three function pointers:
* alloc()
* realloc()
* free()

One can built a trivial HAllocator that merely wraps the system calls
malloc(), realloc(), and free()...and, in fact, this has been done,
and is a global named `system_allocator`.

Instead of
```  char * data = (char *) malloc(50);```
one writes
```  char * data = (char *) system_allocator->alloc(&system_allocator, 50);```

One can do more interesting things with allocators, though.  If, for
example, one preallocated (at an OS level) a large block of memory and
then made an allocator that psuedo-allocated only within that block, a
completely leakless shutdown could be ensured by just freeing (at an
OS level) that entire block.

This is what an `HArena` does.  It's a little more complicated than
that overview, in that it supports dynamic growth of the managed
memory pool (implemented by letting blocks be chained), and has a
header in each block that holds statistics.

The lifecycle of a HArena is :

```
  HArena *my_arena = h_new_arena(&system_allocator, 10 * 1024);

  char * data_1 = (char *) h_arena_malloc(my_arena,  50);
  char * data_2 = (char *) h_arena_malloc(my_arena, 100);
  // use data_1 and data_2
  h_arena_free(my_arena, (void*) data_1);

  data_2 = (char *) NULL;
  // at this point the 100 bytes that data_2 used to point at appears to have leaked
  
  h_delete_arena(my_arena);
  // ...but deleting the arena ensures that there are no true OS-level leaks

```


# Privileged arguments


As a matter of convenience, there are several identifiers that
internal anaphoric macros use. Chances are that if you use these names
for other things, you're gonna have a bad time.

In particular, these names, and the macros that use them, are:
- state:
    Used by a_new and company. Should be an HParseState*
- mm__:
    Used by h_new and h_free. Should be an HAllocator*
- stk__:
    Used in desugaring. Should be an HCFStack*

# Function suffixes

Many functions come in several variants, to handle receiving optional
parameters or parameters in multiple different forms.  For example,
often, you have a global memory manager that is used for an entire
program. In this case, you can leave off the memory manager arguments
off, letting them be implicit instead. Further, it is often convenient
to pass an array or va_list to a function instead of listing the
arguments inline (eg, for wrapping a function, generating the
arguments programattically, or writing bindings for another language.

Because we have found that most variants fall into a fairly small set
of forms, and to minimize the amount of API calls that users need to
remember, there is a consistent naming scheme for these function
variants: the function name is followed by two underscores and a set
of single-character "flags" indicating what optional features that
particular variant has (in alphabetical order, of course):

  __a: takes variadic arguments as a void*[] (not implemented yet, but will be soon. 
  __m: takes a memory manager as the first argument, to override the system memory manager.
  __v: Takes the variadic argument list as a va_list


# Memory managers

If the __m function variants are used or system_allocator is
overridden, there come some difficult questions to answer,
particularly regarding the behavior when multiple memory managers are
combined. As a general rule of thumb (exceptions will be explicitly
documented), assume that

   If you have a function f, which is passed a memory manager m and
   returns a value r, any function that uses r as a parameter must
   also be told to use m as a memory manager.

In other words, don't let the (memory manager) streams cross.

# Language-independent test suite


There is a language-independent representation of the Hammer test
suite in `lib/test-suite`.  This is intended to be used with the
tsparser.pl prolog library, along with a language-specific frontend.

Only the C# frontend exists so far; to regenerate the test suites using it, run

    $ swipl -q  -t halt -g tsgencsharp:prolog tsgencsharp.pl \
          >../src/bindings/dotnet/test/hammer_tests.cs

