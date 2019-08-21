# Move Constructor


| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            |                                                                 |
| Review Count:   | 0                                                               |
| Author:         | Razvan Nitu, Andrei Alexandrescu                                |
| Implementation: |                                                                 |
| Status:         | Draft                                                           |


## Abstract

Currently, the DMD compiler does not perform any move operations: rvalues
are typically constructed into a temporary and then blitted to the destination,
eliding the destruction of the source. This leads to the missing of some
optimization opportunities in certain scenarios, that could be easily implemented.
However, if the moving of objects would be implemented, with the current state of
affaires, it would lead to the impossibility of implementing certain programming patterns
like `struct`s that contain internal pointers and `struct`s that register themselves in a
global registry.

DIP1014 [1] details the above mentioned scenarios and proposes a solution in
the form of a postblit-like callback method called `opPostMove`. In parallel with the
development of DIP1014, DIP1018 [2] has uncovered a series of issues that affect the
postblit which ultimately led to the decision of replacing it with a classical copy constructor.
The postblit-like nature of the `opPostMove` function makes it susceptible to the same issues
encountered in DIP1018.

The current DIP highlights the above mentioned issues and proposes the implementation of a
C++ style move constructor as an alternative solution to enable the compiler to perform
move operations in a controlled manner.

## Contents
* [Rationale](#rationale)
* [Prior Work](#prior-work)
* [Description](#description)
* [Breaking Changes and Deprecations](#breaking-changes-and-deprecations)
* [Reference](#reference)
* [Copyright & License](#copyright--license)
* [Reviews](#reviews)

## Rationale

### Move semantics

Even though it is specified that D compilers may perform move operations, DMD
currently never actually moves, although there are situations where it would make
sense to do so (while preserving correctness):

1. Forwarding a parameter passed by value or a local variable to another function:

```d
struct VeryBigStruct { /* 1 GB of data */ }

void gun(VeryBigStruct s);
void main()
{
    VeryBigStruct s = VeryBigStruct(/*some params*/);
    /* do something */
    gun(s);
    /* do other stuff, but s is not accessed anymore */
}
```

In the above situation, `s` was constructed on the stack supposedly initializing 1GB of
data. When `s` is passed to `gun`, a copy of it is going to be performed, allocating and
initializing a new `VeryBigStruct` object. Considering that `s` is no longer used
in the `main` function, the compiler could take advantage of this situation and perform
a move, thus "stealing" the already allocated object. This will result in eliding a copy
constructor call (avoiding the initialization cost) and also a destructor call. It is safe
to do so because `s` is no longer used after it has been moved.

2. Working with uncopyable objects

```d
struct A
{
    @disable this(ref A rhs) {}
}

void fun(A a) {}
void main()
{
    A a;
    fun(a);     // error: A cannot be copied
}
```

The above code will not compile as `A` is uncopyable and `fun` receives its parameter by
value so a copy of `a` should be performed, however, since `fun(a)` is the last access of `a`
it can be safely moved.

If such operations would be implemneted in the compiler, the user would have no means
to control the moving of objects. DIP1014 proposed a solution for this problem, however
it leverages a flawed and soon to be deprecated component: the postblit.

### DIP1014 Issues

DIP1014 proposes a solution based on the postblit semantics: a user may define an
`opPostMove` function which will get called **after** the move was performed. As
shown in DIP1018, typechecking a function which is called after a blitting phase
is problematic because:

* Qualifier conversions between the source and the destination cannot be performed.

* In the case of `const`/`immutable` destinations, fields cannot be properly updated.

Both of the above issues are present in the design of DIP1014, as the following examples
will highlight:

1. The definition of the druntime function `__move_post_blt` has the follwing signature:

```d
void __move_post_blt(S)(ref S newLocation, ref S oldLocation) nothrow if( is(S==struct) )
```

From the above definition it can be deduced that `opPostMove` (that is called internally by
`__move_post_blt`) will get called solely in situations where the source and the destination
have the same type. This makes it so that the following valid code will be rejected:

```d
struct VeryBigStruct
{
    /* 1 GB of data */
    void opPostMove(const ref VeryBigStruct) {}
}

void gun(VeryBigStruct s);
void main()
{
    const VeryBigStruct s = VeryBigStruct(/*some params*/);
    /* do something */
    gun(s);               // s may be moved
    /* do other stuff, but s is not accessed anymore */
}
```

In this situation, the source type is `const D` (`typeof(s)`), while destination type is `D`. Even though the
`opPostMove` function is correctly defined, it will not get called because the signature of
`__move_post_blt` function in druntime does not accomodate the handling of differently qualified
sources and destinations. It seems that the intention in DIP1014 (following the postblit guidelines)
was to handle solely same source-destination types.

2. `opPostMove` is a normal function with a normal typecheck, i.e. `const`/`immutable` `struct` fields
cannot be initialized or modified by it. Consider the following code snippet, extracted from DIP1014:

```d
struct Tracker
{
    static uint globalCounter;
    uint localCounter;
    uint* counter;

    @disable this(this);

    this(bool local)
    {
        localCounter = 0;
        if( local )
            counter = &localCounter;
        else
            counter = &globalCounter;
    }

    void increment()
    {
        (*counter)++;
    }

    void opPostMove(const ref Tracker oldLocation)
    {
        if( counter is &oldLocation.localCounter )
            counter = &localCounter;
    }
}
```

The above code sample implements the `opPostMove` operator for mutable destinations only, however
if we want to use `immutable struct` destination instances we need to define another overload:

```d
void opPostMove(immutable ref Tracker oldLocation) immutable
{
    if( counter is &oldLocation.localCounter )
        counter = &localCounter;
}
```

The above overload will fail to compile as `opPostMove` is a normal function (not a constructor)
and `counter` is viewed as an `immutable` field, therefore it cannot be modified. One solution
might be to typecheck `opPostMove` as if it were a constructor, but this is problematic due
to the state of the field (raw or cooked) after the blitting phase (consult DIP1018 for an extensive
explanation).

Although point (1) may be mitigated by changing the signature of `__move_post_blt`, point (2) cannot be
solved, as discussed in DIP1018.

### Introducing the Move Constructor

DIP1014 makes a strong argument on the necessity of defining and implementing move semantics,
but its design does not take into account the problems that were discovered with the postblit
during the development of DIP1018, thus making it liable of the same issues.

In this context, the current DIP proposes the implementation of a move constructor, that will
bring the following benefits, as opposed to DIP1014:

 * the feature is used to good effect in the C++ language [3];
 * being a constructor, it will allow updating of `const`/`immutable` fields;
 * it will use the same pattern as the copy constructor implementation, which has succesfully
   replaced the postblit;
 * does not require any druntime changes; the feature will be implemented entirely in the compiler;

## Description

This section discusses all the technical aspects regarding the semantics of the move constructor.

### Syntax

Inside a `struct` definition, a declaration is a move constructor declaration if it is a constructor
declaration that takes the first parameter by value (non-default) and the type of the parameter is of
the same type as the `struct` being defined. Additional parameters may follow if and only if all have
default values. Declaring a move constructor in this manner has the advantage that no parser
modifications are required, thus leaving the language grammar unchanged. Example:

```d
import std.stdio;

struct A
{
    this(A rhs) { writeln("x"); }                     // move constructor
    this(A rhs, int b = 7) immutable { writeln(b); }  // move constructor with default parameter
}
```

Type qualifiers may be applied to the parameter of the move constructor and also to the function
itself, in order to allow defining moves across objects of different mutability levels. The
type qualifiers are optional.

### Semantics

This section discusses all the aspects of the semantic analysis and interaction between the move
constructor and other language features.

#### Move Constructor Usage

A call to the move constructor is implicitly inserted by the compiler whenever a `struct` variable
is moved in memory: whevener an rvalue is passed as a function parameter, the move constructor will
be called.

When returning from a function, the result is either an rvalue or an lvalue. If the result
is an rvalue, in all situations RVO is going to be performed; if the result is an lvalue,
then either NRVO is going to be applied, or the copy constructor is going to get called.
If NRVO cannot be performed, it means that the returned object is referenced from outside
the function so it also cannot be moved.

#### Overloading

The move constructor can be overloaded with different qualifiers applied to
the parameter (moving from a qualified source) or to the move constructor
itself (move to a qualified destination):

```d
struct A
{
    this(A another) {}                        // 1 - mutable source, mutable destination
    this(immutable A another) {}              // 2 - immutable source, mutable destination
    this(A another) immutable {}              // 3 - mutable source, immutable destination
    this(immutable A another) immutable {}    // 4 - immutable source, immutable destination
}
```

The proposed model enables the user to define the move from an object of
any qualified type to an object of any qualified type: any combination of
two among mutable, `const`, `immutable`, `shared`, `const shared`.

The inout qualifier may be applied to the move constructor parameter in
order to specify that mutable, `const`, or `immutable` types are treated the same.

#### Typechecking

The move constructor type-check is identical to that of the constructor [3][4].

Move constructor overloads can be explicitly disabled:

```d
struct A
{
    int* p;
    @disable this(A rhs) {}
    this(immutable A rhs) {}
    A opAssign(A rhs) {}
}


void main()
{
    A a;
    a = fun();
}
```

In order to disable move construction, all move constructor overloads need to be disabled.
Note that the line `a = fun();` is lowered to `a.opAssign(fun());`, but it will still result
in a compilation error since the result of `fun` needs to be move constructed into `opAssign`'s
parameter and that specific move constructor is disabled.

#### Move Constructor vs. Implicit Move

Whenever a move constructor is defined for a `struct`, all implicit moving is disabled:

```d
struct A
{
    int* p;
    this(A rhs) {}
}

void fun(immutable A);

void main()
{
    immutable A a;
    fun(a);           // error: cannot call move constructor type (immutable A) immutable
                      // this is a the last access of a, so it can be treated as an rvalue
}
```

#### Generating Move Constructors

A move constructor is generated implicitly by the compiler for a `struct S` if all of the
following conditions are met:

1. `S` does not explicitly declare any move constructors;
2. S defines at least one direct member that has a move constructor, and that member is
not overlapped (by means of `union`) with any other member.

If the restrictions above are met, the following move constructor is generated:

```d
this(inout(S) src) inout
{
    foreach (i, ref inout field; src.tupleof)
        this.tupleof[i].moveConstructor(field);
}
```

Note that the above code is going to access each field in the source exactly once,
this means that each can be treated as an rvalue, thus calling the move constructor.
If a specific field does not define a move constructor the generated move constructor
is going to fail to typecheck and it will be annotated with `@disable`.

### Perfect Forwarding

The concept of perfect forwarding [6] can be described as forwarding the arguments
of one function to another as though the wrapped function had been directly called.
In C++, it is typically implemented with the use of universal references [7] and the
`std::forward` function in order to preserve the lvalueness/rvalueness of the function
arguments. In D, the closest equivalent to universal references is the `auto ref`
parameter concept:

```d
void fun(T)(auto ref T a) {}

void main()
{
    int a;
    fun(a);   // lvalue
    fun(42);  // rvalue
}
```

However, the limitation in this situation is that the `fun` function parameter
is always going to be an lvalue, regardless of the instantiation value type.
This is problematic for perfect forwarding because if `fun` is called with
an rvalue, the rvalueness cannot pe forwarded to the wrapped function.

The solution proposed by this DIP is to identify via a dataflow analysis algorithm the
last access of a by value parameter/local variable and handle it as an rvalue access.
This simple enhacement does not only solve the perfect forwarding problem but also it
makes the code in certain situations more efficient without any user intervention.

#### Dataflow Analysis Algorithm

The access of a variable `x` is considered the last access in a function if :

  * `x` is part of a `return` statement;
  * there are no other statements that access `x` until the end of the function
    and no gotos pointing to a label that precedes the access of `x`;

The algorithm that identifies the last access of `x` is the following:

  * Each statement in the function body is analyzed in the order of appearance
    in user code;
  * If a statement does not access `x` it is skipped;
  * If a statement does access `x`, it will be marked as the new last access,
    while the previous marked statement will be cleared. If the previous last access
    was a `return` statement, then it will not be cleared. `return` statements that
    access `x` are always the last access of it.
  * `while`/`for`/`do-while`/`foreach` statements that contain accesses of `x`
    will not be considered candidates for last access, except if the accesses
    are inside `return` statements;
  * If a statement that accesses `x` is preceded by a label and between the mentioned
    statement and the end of the function scope there is a goto pointing to the mentioned
    label, the access will not be considered a candidate for last access;

Example:

```d
void gun(int);
void sun(int);
void run(int);
int bun(int);

int fun(int x)
{
    if (x < 2)
        gun(x);
    else
        sun(x);

label1:
    if (bun(x) < 0)
        goto label1;

    if (bun(x) > 5)
        goto label2;

    run(x);
label2:
    return x;
}
```

In the above example, the first use of `x` is in the line `x < 2`. The statement is then considered,
for the moment, the last access of `x` and it will be marked as such. Next, the bodies of the `if`
and the `else` branch will be considered: both access `x`, but since the execution is going to take
one path or the other, both accesses are considered to be the last access, thus clearing `x < 2`.
When `bun(x) < 0` is encountered, it will be considered as a candidate for last access, but only if there
are no `goto`'s targeting `label1` until the end of the function scope. When `goto label1` is encountered,
`bun(x) < 0` is dropped from the list of candidates. At this point, there is not statement that we can
safely consider as being the last access of `x`. The analysis continues with `if(bun(x) > 5`, which
becomes the new candidate; the statement `goto label2` is ignored, because gotos that jump in the
front do not affect the algorithm. When `run(x)` is encountered, it will become the new last access,
while clearing the previous one. Finally, when `return x` is reached it becomes the last access and
it cleares `run(x)` from the list of candidates.

`e1 || e2` and `e1 && e2` expressions are treated the following way:

  * if `e1` accesses `x` and `e2` does not, then `e1` is the last access of `x`;
  * if `e2` accesses `x` and `e1` does not, then `e2` is the last access of `x`;
  * if both `e1` and `e2` access `x`, then `e2` is the last access of `x`;

If the `||` and `&&` operators are present in the condition or body of repetitive instructions
(`for`, `while` etc.) the acceese to `x` will not be considered last access candidates.

Whenever the address of an rvalue parameter/local variable is taken it will lose the
possibility of being treated as an rvalue upon its last access. If the user knows
that an access is the last access of an lvalue, it can use the `move` function in
`std.algorithm.mutation`, thus transforming the lvalue into an rvalue.


### Limitations

## Breaking Changes and Deprecations

## Reference

[1] https://github.com/dlang/DIPs/blob/master/DIPs/accepted/DIP1014.md

[2] https://github.com/dlang/DIPs/blob/master/DIPs/accepted/DIP1018.md

[3] https://en.cppreference.com/w/cpp/language/move_constructor

[4] https://dlang.org/spec/struct.html#struct-constructor

[5] https://dlang.org/spec/struct.html#field-init

[6] https://cpppatterns.com/patterns/perfect-forwarding.html

[7] https://isocpp.org/blog/2012/11/universal-references-in-c11-scott-meyers

## Copyright & License

Copyright (c) 2019 by the D Language Foundation

Licensed under [Creative Commons Zero 1.0](https://creativecommons.org/publicdomain/zero/1.0/legalcode.txt)

## Reviews
