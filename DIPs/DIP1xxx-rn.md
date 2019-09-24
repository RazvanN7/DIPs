# Move Semantics


| Field           | Value                                                           |
|-----------------|-----------------------------------------------------------------|
| DIP:            |                                                                 |
| Review Count:   | 0                                                               |
| Author:         | Razvan Nitu, Andrei Alexandrescu                                |
| Implementation: |                                                                 |
| Status:         | Draft                                                           |


## Abstract

Currently, the DMD compiler automatically performs move operations in specific situations.
The default move operation is to simply copy from one memory location to another. This approach
has the shortcomming that objects with internal pointers cannot be correctly handled.

DIP1014 [1] details the above mentioned scenario and proposes a solution in
the form of a postblit-like callback method called `opPostMove`. In parallel with the
development of DIP1014, DIP1018 [2] has uncovered a series of issues that affect the
postblit, which ultimately led to the decision of replacing it with a classical copy constructor.
The postblit-like nature of the `opPostMove` function makes it susceptible to the same issues
encountered in DIP1018.

The current DIP highlights the above mentioned issues and proposes the implementation of a
C++ style move constructor/move assign operator as an alternative solution to enable the
compiler to perform move operations in a controlled manner.

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

Consider the following code snippet:

```d
struct S
{
   long f;
   long* p;
}

S get()
{
    S r;
    r.p = &r.f;
    return r;
}

void fun(S a)
{
    assert(a.p == &a.f);
}

void main()
{
   fun(get());   // need for move constructor
   S a;
   a = get();    // need for move assignment operator
   assert(a.p == &a.f);
}
```

The natural expectation is that the asserts should pass, because the returned objects
from `get` have had their pointers to explicitly point to the internal field, however,
due to the moving that happens behind the scenes (that the user cannot control)
`f` is now located at a different address and the asserts fails. This makes it
impossible to consistently use objects that contain internal pointers.


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
struct D
{
    void opPostMove(const ref D) {}
}

const(D) fun();
void gun(D s);
void main()
{
    gun(fun());
}
```

In this situation, the source type is `const D` (`typeof(fun())`), while the destination type is `D`.
Even though the `opPostMove` function is correctly defined, it will not get called because the signature
of `__move_post_blt` function in druntime does not accomodate the handling of differently qualified
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

### Introducing move semantics

DIP1014 makes a strong argument on the necessity of defining and implementing move semantics,
but its design does not take into account the problems encountered with the postblit
during the development of DIP1018, thus making it liable of the same issues.

In this context, the current DIP proposes the implementation of a move constructor and a move assignment
operator that will bring the following benefits:

 * a similar feature is used to good effect in the C++ language [3];
 * the move constructor will allow updating of `const`/`immutable` fields;
 * it uses the same pattern as the copy constructor implementation, which has succesfully
   replaced the postblit;
 * does not require any druntime changes; the feature will be implemented entirely in the compiler;

## Description

This section discusses all the technical aspects regarding the semantics of the move constructor/move assign operator.

### Syntax

Inside a `struct` definition, a declaration is a move constructor declaration if it is a copy constructor
declaration and the first parameter is annotated with `@rvalue`:

```d
import std.stdio;

struct A
{
    this(@rvalue ref A rhs) {}                       // move constructor
    this(@rvalue ref A rhs, int b = 7) immutable {}  // move constructor with default parameter
}
```

Inside a `struct` definition, a declaration is a move assign operator declaration if it is an assignment
operator declaration that receives its parameter by `ref`, the parameter is of the same unqualified type
as the `struct` containing it and the parameter is annotated with @rvalue:

```d
import std.stdio;

struct A
{
    void opAssign(@rvalue ref A rhs) {}                   // move assignment operator
    void opAssign(@rvalue A rhs, int b = 7) immutable {}  // move assignment operator with default parameter
}
```

Type qualifiers may be applied to the parameter of the move constructor/move assign operator and also
to the function itself, in order to allow defining moves across objects of different mutability levels. The
type qualifiers are optional.

### Semantics

This section discusses all the aspects of the semantic analysis and interaction between the move
constructor/move assign operator and other language features.

#### Move Constructor Usage

A call to the move constructor is implicitly inserted by the compiler whevener an rvalue is passed
as a function parameter.

When an instance of an object is constructed from an rvalue, the move constructor is not called
because NRVO is performed.

#### Move Assignment Operator Usage

A call to the move assignment operator is implicitly inserted by the compiler whenever an object
is assigned to an rvalue:

```d
struct S
{
    void opAssign(@rvalue ref S) {}
}

S fun();

void main()
{
    S a;
    a = fun();      // move constructor call
}
```

#### Overloading

The move constructor/move assign operator can be overloaded with different qualifiers
applied to the parameter (moving from a qualified source) or to the move constructor/move assign operator
itself (move to a qualified destination):

```d
struct A
{
    this(@rvalue ref A another) {}                        // 1 - mutable source, mutable destination
    this(@rvalue ref immutable A another) {}              // 2 - immutable source, mutable destination
    this(@rvalue ref A another) immutable {}              // 3 - mutable source, immutable destination
    this(@rvalue ref immutable A another) immutable {}    // 4 - immutable source, immutable destination
}
```

The proposed model enables the user to define the move from an object of
any qualified type to an object of any qualified type: any combination of
two among mutable, `const`, `immutable`, `shared`, `const shared`.

The inout qualifier may be applied to the move constructor/move assignment operator parameter in
order to specify that mutable, `const`, or `immutable` types are treated the same.

#### Typechecking

The move constructor type-check is identical to that of the constructor [3][4].
The move assignment operator type-check is identical to that of the `opAssign` function.

Move constructor/move assignment overloads can be explicitly disabled:

```d
struct A
{
    int* p;
    @disable this(@rvalue ref A rhs) {}
    this(@rvalue ref immutable  A rhs) {}
    @disable void opAssign(@rvalue ref A rhs) {}
}


void main()
{
    A a;
    a = fun();
}
```

In order to disable move construction, all move constructor overloads need to be disabled.
Note that the line `a = fun();` is lowered to `a.opAssign(fun());`, but it will still result
in a compilation error because the move assignment operator is disabled.

#### Move Constructor vs. Implicit Move

Whenever a move constructor/move assignment operator is defined for a `struct`, all implicit
move construction/move assignemnt is disabled:

```d
struct A
{
    int* p;
    this(@rvalue ref A rhs) {}
    void opAssign(@rvalue ref A rhs) {}
}

void fun(immutable A);
immutable(A) gun();

void main()
{
    fun(gun());           // error: cannot call move constructor type (immutable A) immutable
    immutable A a;
    a = gun();
}
```

#### Generating Move Constructors/Move Assign Operators

A move constructor is generated implicitly by the compiler for a `struct S` if all of the
following conditions are met:

1. `S` does not explicitly declare any move constructors;
2. S defines at least one direct member that has a move constructor, and that member is
not overlapped (by means of `union`) with any other member.

If the restrictions above are met, the following move constructor is generated:

```d
this(@rvalue ref inout(S) src) inout
{
    static foreach (i, ref inout field; src.tupleof)
    {
        static if (hasElaborateMoveConstructor!(typeof(field)))
            this.tupleof[i].moveConstructor(field);
        else
            this.tupleof[i] = field;
    }
}
```

Note that the above code is going to access each field in the source exactly once,
this means that each can be treated as an rvalue, thus calling the move constructor.
If a specific field does not define a move constructor the generated move constructor
is going to fail to typecheck and it will be annotated with `@disable`.

The generation of move assign operators is done the similarly: the move `opAssign` is called
for every field that defines one.

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

The following section, further describes the situation.

#### Current situation:

Currently, if a function receives an argument by value `void callee(S a)`, the following situations may occur
at caller site:

 * call `callee` with an lvalue: callee(a)   => 1 copy
 * call `callee` with an rvalue: callee(S()) => 1 move

The closest situation to perfect forwarding that one can achieve in D is via `auto ref` parameters:

`void wrapper(T)(auto ref T a) { called(a); }`

At caller site the following situations might occur:

 * call `wrapper` with an lvalue: wrapper(a)    => 0 (by ref) + 1 copy = 1 copy
 * call `wrapper` with an rvalue: wrapper(S())  => 1 move (by val) + 1 copy = 1 move + 1 copy

As it can be observed, when `wrapper` is called with an rvalue it does an extra copy.

If a function receives an argument by ref `void calee(ref S a)`, the following situations may occur at
caller site:

 * call `callee` with an lvalue: callee(a)   => 0
 * call `callee` with an rvalue: callee(S()) => error

Trying to implement perfect forwarding for this function by means of `void wrapper(T)(auto ref T a) { callee(a); }`
yields:

 * call `wrapper` with an lvalue: wrapper(a)   => 0 (by ref) + 0 = 0
 * call `wrapper` with an rvalue: wrapper(S()) => 1 move (by val) + 0 = 1 move

As it can be observed, the difference lies in the fact that when `wrapper` is called with an rvalue
it does a move instead issuing an error.

As an alternative, the standard library offers support for `forward` and `move` functions, where `forward`
is implemented as:

```d
auto ref forward(alias arg)()
{
    static if (__traits(isRef, arg))
    {
        return arg;
    }
    else
    {
        import std.algorithm.mutation : move;
        return move(arg);
    }
}
```

Therefore `wrapper` can be implemented as:

`wrapper(T)(auto ref T a) { callee(forward!a); }`

With this version, we have the following situations:

1. `void callee(S a)`. When `wrapper` is called:

 * with an lvalue: 0 (by ref) + 0 (`forward` returns a ref) + 1 copy (by val to `callee`) = 1 copy.
 * with an rvalue: 1 move (by val) + 1 move (call to `move` which calls the move constructor) = 2 moves.

Here we can observe that an extra move is performed.

2. `void callee(ref S a)`. When wrapper is called:

 * with an lvalue: 0 (by ref) + 0 (`forward` returns ref) + 0 (by ref to `callee`) = 0.
 * with an rvalue: 1 move (by value to `wrapper`) + 1 move (call to `move` which calls the move constructor) = 2 moves.

Again, we have 2 moves instead of issuing an error.

Using `auto ref` as is has the downfall of converting an rvalue into an lvalue inside the body of the wrapper function.
Even though we may reason if the parameter was provided by an rvalue or an lvalue, this does not change the fact that
the parameter will be treated as an lvalue inside the function body.


#### @rvalue

void wrapper(T)(@rvalue auto ref T a) { callee(a); }

1. `@rvalue` can be used solely with `auto ref` parameters.
2. `@rvalue` does not affect overload resolution.
3. After overload resolution and template instantiation a `@rvalue` parameter can be either `@rvalue T` or `@rvalue ref T`, but the compiler always passes the argument by reference.
4. Inside the function body `@rvalue T` means "a reference to an rvalue", `@rvalue ref T` means "a reference to an lvalue".
5. When calling a function with an `@rvalue T` argument we have the following situations:

   * the callee receives the argument by `ref`; in this situation the compiler will issue an error: "Cannot take a reference to an rvalue"
   * the callee receives the argument by value; in this situation a move is performed
   * the callee receives the argument by `@rvalue auto ref`; in this situation the callee will be instantianted to receive `@rvalue` i.e. a reference to an rvalue.

6. When calling a function with an `@rvalue ref T` argument, the rules are the same as if it was a simple `ref T` argument.

7. When a code is compiled with `-preview=rvaluerefparam`, the semantics do not change because `auto ref` will still bind the rvalue
to the non-ref function (otherwise `auto ref` can simply be translated to `ref`).
8. If the last access rules are implemented, `@rvalue T` parameters are exempt from those rules.
9. `@rvalue T` parameters (or rvalue references) cannot be passed to `ref` parameters, except if the parameter is marked as `@rvalue`.
10. `@rvalue T ` parameters act as rvalues when they are used as function arguments, but they act as lvalues in all other situations. If a `@rvalue` variable is used after it was moved an error is issued.
11. Member variables of `@rvalue` parameters are treated under the same rules as non-`@rvalue` parameters.
The purpose of `@rvalue` is to instruct the compiler that a specific parameter is going to be forwarded to
another function at some point. Whatever actions are taken regarding the fields of an `@rvalue` object must
not alter it before it was forwarded.
12. It is unsafe to take the address of a `@rvalue T` parameter.

Example:

1. void callee(S a):

  * wrapper(a)    => 0 (@rvalue ref S) + 1 copy = 1 copy
  * wrapper(S())  => 0 (@rvalue S)     + 1 move (rule 5b) = 1 move

2. void callee(ref S a):

  * wrapper(a)   => 0 (@rvalue ref S) + 0 (by ref) = 0
  * wrapper(S()) => 0 (@rvalue S)     + error(rule 5a)    = error




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
