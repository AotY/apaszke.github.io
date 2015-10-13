---
layout: post
title:  "A quick tour of Torch internals"
date:   2015-10-12 20:05:24
---

Recently I have been kind of confused. I couldn't find myself
anything to work on and had no ideas for new projects (apparently I had to wait
for the new academic year to start, when I don't have much time to work on them).

Anyway, I often get the impression that many people are using Machine
Learning libraries as kind of black-boxes with only a high-level API. It's as if they
weren't interested in how they work, but solely in the output (this is why I like
torch so much - it's hackable to the bone). I've been using torch for a few months
now and I've always been curious how it's built. This is why I decided to get
down to it and browse the code of TH library, which is at the core of torch.

It's really a great thing to do. I'll write more about it in the end of this post,
but you should seriously consider doing this too, with your favourite library
or framework.

Torch's source is written in plain C, which was very pleasing for me. I don't
really like many C++ features and although I find it very powerful and flexible,
it often seems confusing. C's extremely minimal syntax allows you to read and quickly
grasp what exactly happens at any moment.

## Where can you get it?

You can find the `TH` library in two places:

* [In it's standalone repository](https://github.com/torch/TH) (git subtree of torch7; outdated at the time of writing)
* [In torch7 repository](https://github.com/torch/torch7) in `lib/TH` folder (always up to date)

The folder structure is very simple. There are some cmake tests and definitions
in `cmake` folder while the code is located both in `general` folder and at the repo root.

## Interesting findings

Before going into the details and describing functionality implemented in individual
files I'd like to point out some really cool techniques that I've found in the implementation.

### Code generation

First thing that appeared really strange was that many files existed both in the
root folder as well as in `generic`. If you opened them you would quickly notice
that copies in `general` contain the actual code, while at the root they all look
very similar. Here is `THStorage.c` for example:

{% highlight c %}
#include "THAtomic.h"
#include "THStorage.h"

#include "generic/THStorage.c"
#include "THGenerateAllTypes.h"

#include "generic/THStorageCopy.c"
#include "THGenerateAllTypes.h"
{% endhighlight %}

Quite unusual for a `.c` file, right?

`THGenerateAllTypes` sounds interesting so i looked it up and this if what I've
found:

{% highlight c %}
#ifndef TH_GENERIC_FILE
#error "You must define TH_GENERIC_FILE before including THGenerateAllTypes.h"
#endif

#define real unsigned char
#define accreal long
#define Real Byte
#define TH_REAL_IS_BYTE
#line 1 TH_GENERIC_FILE
#include TH_GENERIC_FILE
#undef real
#undef accreal
#undef Real
#undef TH_REAL_IS_BYTE

#define real char
#define accreal long
#define Real Char
#define TH_REAL_IS_CHAR
#line 1 TH_GENERIC_FILE
#include TH_GENERIC_FILE
#undef real
#undef accreal
#undef Real
#undef TH_REAL_IS_CHAR
...
{% endhighlight %}

Which continued for a few more types. At first I was puzzled, but then I suddenly
realized what it does and how brilliant this is! There are no templates in C, but
objects like THStorage should be available for different types. It would be a
terrible waste to repeat the same implementation with just a few words replaced
and this is what this piece archieves! In generic files you can see
variables of type `real` all over the place. At first it was obvious to me that
it's probably a matter of some compile time optimizations wether it was chosen to
be a float or a double, but apparently it's different - it allows code generation
for other types!

<div style="height: 17px"></div>

Clever use of macros also makes the generic files more readable.
Take this example from `generic/THStorage.c`:

{% highlight c %}
real* THStorage_(data)(const THStorage \*self)
{% endhighlight %}

It looks nice, but what about name conflicts for different types? It can't be `THStorage`
and `THStorage_data` all the time! Worry not, macros take care of that as well:

{% highlight c %}
#define THStorage        TH_CONCAT_3(TH,Real,Storage)
#define THStorage_(NAME) TH_CONCAT_4(TH,Real,Storage_,NAME)
{% endhighlight %}

During preprocessing this function name will be expanded to something like `THByteStorage_data` and
`THStorage` will be replaced with `THByteStorage`. Super cool x2!

<div style="height: 17px"></div>

It's also smart to use a `#line 1 TH_GENERIC_FILE` directive, because if there
would be any errors they will appear in the compiler as if they were in the original
generic file - not in the middle of the implementation pasted over and over.

I think that these are some awesome ways to make C code more type-agnostic.

### OOP & Virtual tables

TH also implements a file API, where you can find good examples of how you could
implement some basic OOP patterns in C. There are four files that I'll be
talking about here:

* `THFilePrivate.h` - defines basic structs
* `THFile.c` - contains some generic implementation
* `THDiskFile.c` - code for handling disk files
* `THMemoryFile.c` - implementation of in-memory files

Let's start with the private header file.

{% highlight c %}
struct THFile__
{
    struct THFileVTable \*vtable;

    int isQuiet;
    ...
};

struct THFileVTable
{
    int (\*isOpened)(THFile \*self);

    long (\*readByte)(THFile \*self, unsigned char \*data, long n);
    long (\*readChar)(THFile \*self, char \*data, long n);
    long (\*readShort)(THFile \*self, short \*data, long n);
    long (\*readInt)(THFile \*self, int \*data, long n);
    ...
};
{% endhighlight %}

You can see that it defines a virtual table with methods that File subclasses will
have to implement, because `THFile` is an abstract class (it has no constructors).
Other structs are defined as such:

{% highlight c %}
typedef struct THDiskFile__
{
    THFile file;

    FILE \*handle;
    char \*name;
    int isNativeEncoding;

} THDiskFile;
{% endhighlight %}

What makes it interesting is that because it's first member is of type THFile
it's actually valid to cast `struct THDiskFile *` to `struct THFile *` and use
it normally. What's more, because `THDiskFile`'s constructor changes the function pointers
in `file` field's virtual table, it will behave as `THDiskFile` object even when casted to `THFile`!

### Shared memory

I had little knowledge about UNIX process management and threading until now, when
I took up an operating systems course at my university, so it was really interesting
to learn about `mmap` (maps a file to memory, so you can use it like an array)
and to see how memory can be shared between processes with `shm_open`. I even
wrote a piece of code to try it out. You can find it [in here](https://github.com/apaszke/apaszke.github.io/tree/master/assets/posts/torch-internals/shared_mem).

### SIMD

Another cool thing you can find in `TH` are vector instructions.
There are some cmake tests that check if they are available on your CPU
(`cmake/FindSSE.cmake`) and several files implementing convolution operations
using them (`generic/simd/*`). I can't understand it yet - function that takes
10 lines of code is expanded to unrolled vectorized loop taking more than 120
lines and using APIs with unreadable function names for a SSE beginner.
This code spans 134 lines after macro expansion:

{% highlight c %}
void convolve_5x5_1_avx(float* output, float* image, float* weight, long count, long outputStride, long inputStride) {
  long i = 0;
  long alignedCount = count & 0xFFFFFFF8;
  DECLARE_OUTPUT_1()
  for (; i < alignedCount; i+=8) {
    CONVOLVE_8COLS_XROWS(1, i)
  }
}
{% endhighlight %}

Anyway, it's definitely a thing worth learning so I will probably write more about it soon!

### Allocators

`TH` declares it's own function for memory allocation called `THAlloc`. It tries
to allocate a properly aligned chunks if you allocate big blocks and handles
out-of-memory errors. Before reading torch's source I didn't know about the
concept of allocators. It's cool that you can pass an Allocator to `THStorage`
or `THTensor` and construct it not only in the regular heap region, but also
allocate it in shared memory.

### Random module

It's natural to have a pseudorandom number generator in all languages, but I've
never read an implementation of one (ok, except [linear congruential generator](https://en.wikipedia.org/wiki/Linear_congruential_generator)). In
`THRandom.c` you can find a full implementation of [Mersenne twister](https://en.wikipedia.org/wiki/Mersenne_Twister),
which (according to Wikipedia) is a default implementation for R, Python, Ruby,
PHP, CMU Common Lisp, GLib, MATLAB and some more. There are also several methods
which convert returned uniform distribution into other shapes.

## Quick library overview

In this section I will briefly describe most of the functionalities provided by `TH`.

* **THAllocator**
    * creates a default default allocator, which just calls `TH` memory
    management functions and, if possible, a `THMapAllocator` that can map files
    or shared memory objects into memory.
* **THAtomic**
    * multiplatform implementation of atomic operations
* **THTensor**
    * defines a general Tensor type
    * supports lots of indexing, linear algebra and general math operations
    * available for all primitive datatypes (`TH<type>Tensor`, e.g. `THFloatTensor`)
* **THBlas**
    * wraps BLAS library for use in `THTensor`
    * provides a general implementation as a fallback
* **THLapack**
    * wraps LAPACK library for use in `THTensor`
    * **DOESN'T** provide fallbacks - throws errors if called
* **THFile**
    * abstract file class
    * only creates wrappers for calling methods contained in virtual table
* **THDiskFile**
    * concrete file class
    * wraps disk file APIs
* **THMemoryFile**
    * concrete file class
    * operates on an in-memory buffer and fakes file operations
* **THGeneral**
    * implements general utilities
    * contains memory management routines
    * can hook to an external GC
* **THRandom**
    * implements a random number generator
    * can sample from many distributions
* **THStorage**
    * defines a general storage object
    * contains mainly bookkeeping code
    * available for all primitive datatypes (`TH<type>Storage`, e.g. `THFloatStorage`)

## How to use it

> This section will be expanded soon

## Example program

> This section will be expanded soon

## Afterthoughts

I actually enjoy reading other's source code -
especially if it's well written. If you have some time, then seriously, consider
picking your favourite library or framework and try to understand how it works, even
the tiniest bits of it. I guarantee that you will find many fascinating things
and learn many concepts and ways of structuring your code that you had no idea existed.
I haven't learned that much in such short period of time for a while. I liked
it so that I'm thinking about doing this on a more regular basis.