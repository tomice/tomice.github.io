---
title: "Undefined Behavior vs Expected Behavior"
description: GCC 15 changes historical behavior
date: 2025-04-27 00:00:00 -0400
categories: [Programming, C, C++, GCC, Clang, Compilers]
tags: [programming, c, c++, compilers]
---

The [GNU Compiler Collection](https://gcc.gnu.org/) recently released
[GCC 15](https://gcc.gnu.org/gcc-15/) which brings a ton of features to C and
C++ such as having C23 be the default, supporting `#embed`, adding various C++26
features like variadic friends, attributes for structured bindings, and more.

However, one thing specifically stood out, and it was at the very beginning of
the caveats section in the [changelog](https://gcc.gnu.org/gcc-15/changes.html).
Third bullet point down, we see the following:

> `{0}` initializer in C or C++ for unions no longer guarantees clearing of the
> whole union (except for static storage duration initialization), it just
> initializes the first union member to zero. If initialization of the whole
> union including padding bits is desirable, use `{}` (valid in C23 or C++) or
> use `-fzero-init-padding-bits=unions` option to restore old GCC behavior.

This caveat, listed literally half way in the list of caveats for this release
is causing a bit of an uproar in the community, and for good reason. Let's
explore why people are upset, the counter arguments the GCC team has, and why
this is a perfect example between what is considered undefined behavior and
what is considered expected behavior based on historical behavior.

## Undefined Behavior

Before we begin, let's take a stroll down what undefined behavior is and why it
exists in the first place. Both C and C++ have a concept called undefined
behavior that is defined by the
[C99 ISO spec](https://www.dii.uchile.cl/~daespino/files/Iso_C_1999_definition.pdf)
as:

> Behavior, upon use of a nonportable or erroneous program construct or of
> erroneous data, for which this International Standard imposes no
> requirements.
>
> NOTE: Possible undefined behavior ranges from ignoring the situation
> completely with unpredictable results, to behavior during translation or
> program execution in a documented manner characteristic of the environment
> (with or without the issuance of a diagnostic message), to terminating a
> translation or execution (with the issuance of a diagnostic message).

For some, those lines might be a bit hard to follow, but the
[C FAQ](https://c-faq.com/ansi/undef.html) sums it up nicely:

> Anything at all can happen; the Standard imposes no requirements.
> The program may fail to compile, or it may execute incorrectly (either
> crashing or silently generating incorrect results), or it may fortuitously
> do exactly what the programmer intended.

So essentially, when a program exhibits undefined behavior, the C standard
imposes no limits or barriers on what the compiler itself must do. It truly
can do whatever it wants. The compiler team doesn't even need to document
it. It doesn't need to be safe. The world is their oyster! For those coming
from a language like Java where everything is defined, this is probably a wild
concept, but it's a commonly understood paradigm in the C and C++ world.

C and C++ developers work around undefined behavior on a daily basis. Allowing
the concept of undefined behavior is almost like having a built-in optimizer.
The compiler is not forced to perform overhead operations such as checking if a
variable is defined or not, if signed integer overflow is occurring, if there
is out-of-bounds array access, and so on. I won't get into whether this is a
good thing or bad thing. Many other blogs touch on this in much greater
detail than I could every go into. I listed some great ones at the end you
can check out.

## The "Other" Behaviors

Alongside the undefined behavior section in the ISO C99 specification is a
section called unspecified behavior. The standard defines it as:

> Behavior where this International Standard provides two or more possibilities
> and imposes no further requirements on which is chosen in any instance.
>
> EXAMPLE: An example of unspecified behavior is the order in which the
> arguments to a function are evaluated.

Summed up, the compiler team is free to do whatever they want in certain
scenarios when they face unspecified behavior, and they never need to report it
or document it in any way.

There are two more defined behaviors in the spec worth touching on briefly:

> Implementation-defined behavior: unspecified behavior where each
> implementation documents how the choice is made
>
> EXAMPLE: An example of implementation-defined behavior is the propagation
> of the high-order bit when a signed integer is shifted right.

and

> Locale-specific behavior: behavior that depends on local conventions of
> nationality, culture, and language that each implementation documents.
>
> EXAMPLE: An example of locale-specific behavior is whether the `islower`
> function returns true for characters other than the 26 lowercase Latin
> letters.

So for implementation-defined behavior, the implementor must pick some possible
behavior, verify it does not fail to compile, and the choice that the
implementor chooses should be documented in some way. This makes sense as it
means there are multiple avenues you are free to take; just choose one and make
sure those who are using your compiler know about it. Locale-specific behavior
also makes sense as not everyone shares the same language, and the standard
does not want to impose restrictions based off of a specific language.

## The "Other...Other" Behavior

Finally, we get to the crux of the issue, and that is one I am defining in this
blog. One that I am calling "historical behavior." I am purposely being a bit
fast and loose here, as this is not actually a true behavior defined in the
spec, nor is it an agreed upon concept. However, I do feel it is an important
one that should be considered beyond what the spec itself points out.

Historical behavior can be thought of as follows:

> Historical behavior refers to cases where undefined or unspecified behavior,
> although never formally guaranteed by the language standard, has been handled
> in a consistent and predictable way by a major compiler implementation for
> many years - to the point where developers have come to rely on it as though
> it were guaranteed. Over time, this stability leads to the assumption that the
> behavior is effectively "safe," even though no formal guarantee ever existed.
>
> EXAMPLE: `{0}` initializer in C or C++ for unions guarantees clearing of the
> whole union, including padding bits, in order to maintain developer
> expectations based on long-term historical compiler behavior despite not
> being explicitly defined by the specification.

Now, I am not a standards committee member, nor am I one that tends to dive
head first into technical volume writing like many other standards writers
are. I am sure this definition isn't necessarily perfect and can be argued
that it is negated by the very existence of the aforementioned behaviors.

I do think it is worth considering, however. In fact, I believe that even
some compiler team members feel that way, as we will look into how the Clang
team decided to handle this very behavior in a moment.

## Type Punning

Before diving into the heart of the issue, we first need to discuss type
punning and what it means, as this is where we will observe this new change
break.

Type punning is the act of treating a piece of memory as if it were a different
type. Unions have historically been used for type punning both in C and C++
(although less with C++, especially as it moves into modern C++), even though
there are some caveats around strict aliasing and undefined behavior.

Consider a situation where you are working with a graphical application and
need to interpret pixel data, which is usually stored as integers, but needs to
be interpreted as floating-point numbers for certain calculations (first one
that comes to mind is gamma correction).

Using type punning via unions, you could directly access the bit patterns of
the color values, potentially saving on computation time and memory usage,
while still having access to them as both integers and floats for different
operations.

```c
#include <stdio.h>

union Color {
    int rgba;            // 32-bit integer
    float rgba_float[4]; // Same storage interpreted as 4 floats
};

int main(void) {
    // Clear all 4 floats to 0.0
    union Color color = {0};

    // Set red channel to 1.0 (RGBA: 1.0, 0.0, 0.0, 0.0)
    color.rgba_float[0] = 1.0f;

    // Print out each channel
    printf("Red:   %f\n", color.rgba_float[0]);
    printf("Green: %f\n", color.rgba_float[1]);
    printf("Blue:  %f\n", color.rgba_float[2]);
    printf("Alpha: %f\n", color.rgba_float[3]);

    return 0;
}
```

This allows you to work with the color data at both the bit level and as
floating-point values without having to copy or cast the data. Cool, right?
As described earlier, this has caveats. Let's take a look at what the C99 and
[C11 ISO](https://www.open-std.org/jtc1/sc22/wg14/www/docs/n1548.pdf)
specifications say about this.

### The C99 and C11 Standard Definition

Buried in Annex J of the C99 standard, we see this listed in the portability
issues section in J.1 Unspecified Behavior

> The following are unspecified:
> The value of a union member other than the last one stored into (6.2.6.1)

As we learned earlier, unspecified behavior basically means anything can happen
and it doesn't need to be documented. The implementor simply must pick some
behavior and run with it. That is exactly what compiler developers did, too.
For type punning via union members, the use of `{0}` was allowed and worked to
fully zero initialize a struct or union. We even see it used in the above code
as I am making sure to zero initialize the union before I ever write any value
into it to avoid garbage data. I only perform that single zero initialization
command, and I continue on my way using the data.

C11, however, decided to become a bit more explicit on this, removed the
statement from Annex J, and stated the following in the 6.5.2.3 Structure and
union members section:

> If the member used to read the contents of a union object is not the same as
> the member last used to store a value in the object, the appropriate part of
> the object representation of the value is reinterpreted as an object
> representation in the new type as described in 6.2.6 (a process sometimes
> called "type punning"). This might be a trap representation.

This means type punning through a union is explicitly permitted in C11,
provided that the data being reinterpreted forms a valid object
representation under the new type. So basically, you are allowed to store a
value into one union member (like an `int`) and then read from another member
(such as a `float`).

However, you must be aware that if the stored bit pattern does not form a valid
value for the new type, it could result in a trap representation which leads to
undefined behavior when accessed.

This change is important because it guarantees that type punning itself is not
undefined behavior according to the actual specification. However, there is a
subtle edge case not mentioned, one being that reading an invalid object
representation resulting from the punning could still cause undefined behavior.

To handle this, devs relied on `{0}` to zero the entire union. This was
considered relatively safe as devs could reasonably assume that type punning
after such a clear would not lead to invalid states. This is because all
padding, floats, integers, and other types would start from a fully zeroed,
valid memory state.

### The C++11 Standard Definition

For completion's sake, let's take a look at what C++ states regarding unions.
We'll take a look at the
[C++14 ISO draft spec](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4296.pdf),
specifically:

> In a union, at most one of the non-static data members can be active at any
> time, that is, the value of at most one of the non-static data members can be
> stored in a union at any time.

This wording introduces the concept of an "active member" in a union. At any
given time, only one member of the union is considered active. Basically, it
means that only one member's value is valid, and accessing any other member is
not permitted unless special rules apply (which are mentioned shortly after
the quoted section above).

Unlike in C, where type punning through unions is decently defined (as long as
object representations are valid), C++ imposes stricter rules. Accessing a
union member that is not currently active is undefined behavior unless very
specific conditions are met.

This rule actually ties into a broader part of the C++ object model found in
the 3.10 lvalues and rvalues section of the C++14 specification. In there, we
see the following:

> If a program attempts to access the stored value of an object through a
> glvalue of other than one of the following types the behavior is undefined:
>
> * The dynamic type of the object,
> * A cv-qualified version of the dynamic type of the object,
> * A type similar (as defined in 4.4) to the dynamic type of the object,
> * A type that is the signed or unsigned type corresponding to the dynamic
>   type of the object,
> * A type that is the signed or unsigned type corresponding to a cv-qualified
>   version of the dynamic type of the object,
> * An aggregate or union type that includes one of the aforementioned types
>   among its elements or nonstatic data members (including, recursively, an
>   element or non-static data member of a subaggregate or contained union),
> * A type that is a (possibly cv-qualified) base class type of the dynamic
>   type of the object,
> * A char or unsigned char type.

Applying this to unions - the "effective type" of the storage is determined by
whichever union member was most recently activated. Attempting to read a
different union member that does not match the active type causes undefined
behavior because it violates the strict effective type rules of C++.

There are some more subtleties, but the main thing I am trying to show is that
C and C++ differ quite a bit in type punning rules, so you cannot assume one
set of rules applies to the other. They are different specs, after all.

## Compiler Wars

The two main compilers we tend to use these days in C are GNU GCC and
[Clang](https://clang.llvm.org/). Almost all of my professional work involves
using GCC. However, the latest news of GNU GCC 15.1 changing the behavior of
zero'ing a union had me check into what Clang is doing, and the most
interesting this is that they almost swapped their own unspecified behaviors.

### Creating an Example

We'll start with the first base-case of GNU GCC and its historic behavior of
zero'ing a union. We'll use this example code:

```c
#include <stddef.h>

union Foo {
    float f;  // 4 bytes
    double d; // 8 bytes
};

// Initializes only the first member (float f)
void init_f(union Foo *u) {
    *u = (union Foo){0};
}

// Initializes the second member (double d)
void init_d(union Foo *u) {
    *u = (union Foo){.d = 0};
}

int main(void) {
    union Foo foo;

    // Call init functions
    init_f(&foo);
    init_d(&foo);
    
    return 0;
}
```

Now let's run this into [godbolt](https://godbolt.org/) and see what happens!

### GCC 11.1

The following is the resulting x86-64 assembly from GCC 11.1. We will focus
on the `init_f` assembly that is generated:

```bash
init_f:
        push    rbp
        mov     rbp, rsp
        mov     QWORD PTR [rbp-8], rdi
        mov     rax, QWORD PTR [rbp-8]
        mov     QWORD PTR [rax], 0
        nop
        pop     rbp
        ret
```

Here, we see `mov QWORD PTR [rax], 0`. This means the full 8-byte store is
being initialized to zero even though we are initializing the float. This
is the expected behavior devs have come to know.

In memory, it would look something like this:

```bash
Memory:
[00][00][00][00][00][00][00][00]
 ^^^^^^^^^^^^^^ first 4 bytes = float f
                 ^^^^^^^^^^^^^^ rest 4 bytes = padding / start of double d

Result:
- Full 8 bytes cleared to 0
- Both f and d are safely 0
- No garbage left anywhere
```

### GCC 15.1

Now, let's take a look at how GCC 15.1 handles it with the new change:

```bash
init_f:
        push    rbp
        mov     rbp, rsp
        mov     QWORD PTR [rbp-8], rdi
        mov     rax, QWORD PTR [rbp-8]
        pxor    xmm0, xmm0
        movss   DWORD PTR [rax], xmm0
        nop
        pop     rbp
        ret
```

Here, we see the following:

```bash
pxor xmm0, xmm0
movss DWORD PTR [rax], xmm0
```

We are clearing XMM0, but we are only storing 4 bytes (single precision), not
the full 8 bytes like we were back with GCC 11.1.

In memory, it would look something like this:

```bash
Memory:
[00][00][00][00][??][??][??][??]
 ^^^^^^^^^^^^^^ first 4 bytes = float f
                 ^^^^^^^^^^^^^^ remaining 4 bytes = untouched

Result:
- First 4 bytes (float f) zeroed.
- Last 4 bytes (upper half of double d) left uninitialized (garbage values).
```

This is the new expected behavior. But let's see what the other compiler team
is doing...

### Clang 6.0.0

Same code, same model. We'll take a look at Clang 6.0.0 first:

```bash
init_f:
        push    rbp
        mov     rbp, rsp
        xorps   xmm0, xmm0
        mov     qword ptr [rbp - 8], rdi
        mov     rdi, qword ptr [rbp - 8]
        movss   dword ptr [rbp - 16], xmm0
        mov     rax, qword ptr [rbp - 16]
        mov     qword ptr [rdi], rax
        pop     rbp
        ret
init_d:       
```

So this is interesting. The real key points are here:

```bash
xorps   xmm0, xmm0     # zero xmm0
movss   [rbp-16], xmm0 # write 4 bytes (float)
mov     rax, [rbp-16]
mov     [rdi], rax     # write 8 bytes
```

Clang writes 4 bytes, but then copies the whole 8 bytes from the stack.
This ends up clearing the entire union, but this is more of a side-effect
than an actual undocumented behavior. This is near what the latest version of
GCC 15.1 is doing. Kinda neat! But also, not what I was expecting thanks to
that little side-effect. So let's see what the very latest version of Clang
does.

### Clang 20.1.0

This is from Clang 20.1.0:

```bash
init_f:
        push    rbp
        mov     rbp, rsp
        sub     rsp, 32
        mov     qword ptr [rbp - 8], rdi
        mov     rax, qword ptr [rbp - 8]
        mov     qword ptr [rbp - 24], rax
        xorps   xmm0, xmm0
        movss   dword ptr [rbp - 16], xmm0
        lea     rdi, [rbp - 16]
        add     rdi, 4
        xor     esi, esi
        mov     edx, 4
        call    memset@PLT
        mov     rax, qword ptr [rbp - 24]
        mov     rcx, qword ptr [rbp - 16]
        mov     qword ptr [rax], rcx
        add     rsp, 32
        pop     rbp
        ret
```

This is doing a lot more under the hood than before. The key point is here:

```bash
xorps   xmm0, xmm0     # zero xmm0
movss   [rbp-16], xmm0 # write 4 bytes (float)

lea     rdi, [rbp-16]      
add     rdi, 4         # start after float
xor     esi, esi       # fill value = 0
mov     edx, 4         # fill 4 bytes
call    memset@PLT     # explicitly zero the upper 4 bytes

mov     rcx, [rbp-16]
mov     [rax], rcx     # copy full 8 bytes
```

Essentially, Clang is zero'ing the 4 byte float, but then it goes ahead and
`memset`'s the rest of the union resulting in behavior similar to the older
versions of GCC. I am guessing this is to make sure the entire memory region
is zeroed out. TO me, it is almost as if the two compiler teams switched
opinions on how this should be implemented with GCC now only clearing the first
member, and Clang now clearing out the entire region.

For anyone who wants to verify this, I created some godbolt links.

* GCC 11.1 vs GCC 15.1: <https://godbolt.org/z/EnWKzcWGz>
* Clang 6.0.0 vs Clang 20.1.0: <https://godbolt.org/z/MMebbveE6>

## Where GCC Breaks Historical Behavior

So here is where we get to the main point of the issue. For over a decade, C
devs have expected `{0}` to fully clear unions. Type punning appears to be
allowed, at least to some degree, in the newer C standards, and compilers have
gone to great lengths to try to maintain memory safety by adhering to expected
behavior, even if it is undefined or unspecified.

The latest change in GCC's handling of this breaks what I consider to be
historical behavior, and it might lead to some interesting bugs that begin to
pop up for those using older codebases.

Those who work entirely on bare metal, fully embedded devices usually do not
change compilers often, so they are mostly safe. Those who work with the very
latest spec and upgrade compilers often also tend to be safe as they tend to
refactor often and write the most up-to-date code.

It is those of us who deal with legacy code and actively port it to newer,
embedded-style OSes with modern compilers that are going to really bear the
brunt of this change. Those of us who are taking enterprise OSes with legacy
code that "worked fine" and are porting it to purpose-built embedded
Yocto-based distros will be the ones to start finding all these fun bugs.

I completely understand why the GNU GCC team is allowed to do this given how
the spec reads. Many GCC devs claim that
[type punning via unions is undefined](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=118141#c13).
There's also no such thing as truly historical behavior in the spec. However,
there are certain assumptions programmers rely on to be valid, for better or
for worse, even if they aren't well-defined. I really do believe this is one of
those cases where I think the compiler team might want to re-evaluate their
decision on making this flag-defined behavior instead of default behavior, or
at the very least present a solid argument as to why they are shifting away
from a historical behavior that many of us have come to rely on, even if it
was considered UB.

## Further Reading

Some useful links on undefined behavior:

* [What Every C Programmer Should Know About Undefined Behavior 1/3](https://blog.llvm.org/2011/05/what-every-c-programmer-should-know.html)
* [A Guide to Undefined Behavior in C and C++, Part 1](https://blog.regehr.org/archives/213)
* [C FAQ Question 11.33](https://c-faq.com/ansi/undef.html)

Happy compiling!
