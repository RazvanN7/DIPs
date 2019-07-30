# Move Constructor


| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            |                                                                 |
| Review Count:   | 0                                                               |
| Author:         | Razvan Nitu, Andrei Alexandrescu                                |
| Implementation: |                                                                 |
| Status:         | Draft                                                           |


## Abstract

DIP1014 [1] states the problem the D programming language currently exhibits with
regard to move semantics and proposes a solution in the form of a postblit-like
callback method called `opPostMove`. In parallel with the development of DIP1014,
DIP1018 [2] has uncovered a series of issues that affect the postblit which ultimately
led to the decision of replacing it with a classical copy constructor.  The postblit-like
nature of the `opPostMove` function makes it susceptible to the same issues encountered in
DIP1018.

The current DIP highlights the above mentioned issues and proposes the implementation of a
C++ style move constructor as an alternative solution.

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

This DIP forwards the rationale presented in DIP1014 for the necessity of defining
move semantics.

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
have the same type or one of the types is implicitly convertible to the other. This makes it
so that the following valid code will be rejected:

```d
struct D
{
    D* p;
    void opPostMove(immutable ref D oldLocation)
    {
        p = oldLocation.getP();
    }

    D* getP() immutable
    {
        return cast()p;
    }
}

immutable(D) fun();

void main()
{
    D d = fun();    // error: `__move_post_blt` does not match template declaration
}
```

In this situation, the source type is `immutable D`, while destination type is `D`. Even though the
`opPostMove` function is correctly defined, it will never get called because the signature of
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

While point (1) may be mitigated by changing the signature of `__move_post_blt`, point (2) cannot be
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

A fun();

void main()
{
    A a;
    A b = fun();           // calls move constructor implicitly - prints "x"
    A c = A(fun());        // calls constructor explicitly
    immutable A d = fun(); // calls move constructor implicittly - prints 7
}
```

The move constructor may also be called explicitly (as shown above in the line introducing c)
because it is also a constructor within the preexisting language semantics.

Type qualifiers may be applied to the parameter of the move constructor and also to the function
itself, in order to allow defining moves across objects of different mutability levels. The
type qualifiers are optional.

### Semantics

This section discusses all the aspects of the semantic analysis and interaction between the move
constructor and other language features.

#### Move Constructor Usage

A call to the move constructor is implicitly inserted by the compiler whenever a `struct` variable
is initialized from an rvalue of the same unqualified type:

1. When a variable is explicitly initialized:

```d
struct A
{
    this(A rhs) {}
}

A fun();

void main()
{
    A a;
    A b = fun();     // move constructor gets called
    b = fun();       // assignment, not initialization
}
```

2. When a parameter is passed by value to a function:

```d
struct A
{
    this(A rhs) {}
}

A gun();
void fun(A a);

void main()
{
    A a;
    fun(gun());    // move constructor gets called
}
```

When a function returns an rvalue there is no need to call the move constructor
as RVO (return value optimization) will be performed and the resulting `struct`
instance will be constructed in the caller stack.

Note that calls to the move constructor will not be implicitly inserted for constructor
calls:

```d
struct A
{
    this(A rhs) {}
    this(int b) {}
}

void main()
{
    A a;
    A b = S(a);      // move constructor is called explicitly, no need to insert another call
    A c = S(7);      // constructor call, no need to insert move constructor call
}
```

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

A fun();
immutable(A) gun();

void main()
{
    A a = fun();                    // calls 1
    A b = gun();                    // calls 2
    immutable A c = fun();          // calls 3
    immutable A d = gun();          // calls 4
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
    void opAssign(A rhs) {}
}

A fun();
immutable(A) gun();

void main()
{
    A a = fun();  // error: disabled move constructor
    A b = gun();  // ok
    a = fun();    // opAssign is used, but still error

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

immutable(A) fun();

void main()
{
    immutable A a = fun();       // error: cannot call move constructor type (immutable A) immutable
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
        this.tupleof[i] = field;
}
```

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
last access of an rvalue parameter/local variable and handle it as an rvalue access.
This simple enhacement does not only solve the perfect forwarding problem but also it
makes the code in certain situations more efficient without any user intervention.

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
