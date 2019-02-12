# `__mutable` storage class

| Field           | Value                                                                           |
|-----------------|---------------------------------------------------------------------------------|
| DIP:            | (number/id -- assigned by DIP Manager)                                          |
| Review Count:   | 0 (edited by DIP Manager)                                                       |
| Authors:        | Timon Gehr - timon.gehr@gmx.ch, Razvan Nitu - razvan.nitu1305@gmail.com         |
| Implementation: | (links to implementation PR if any)                                             |
| Status:         | Will be set by the DIP manager (e.g. "Approved" or "Rejected")                  |

## Abstract

This document proposes the addition of a new storage class, `__mutable`, that
may be used to tag function declarations or aggregate declaration member fields.

### Reference

* [1] https://github.com/dlang/phobos/blob/master/std/typecons.d#L6153

* [2] https://github.com/dlang/phobos/blob/master/std/container/array.d#L430

* [3] https://github.com/dlang/phobos/blob/master/std/typecons.d#L2126

* [4] https://github.com/dlang/phobos/blob/master/std/datetime/systime.d#L9569

* [5] https://dlang.org/spec/function.html#pure-functions


## Rationale and Motivation

This section discusses compelling arguments on the necessity of the `__mutable` storage class.

### The necessity of `__mutable` for aggregate declaration member fields

Type qualifiers are applied transitively to all subtypes:

```d
struct A
{
    int a;
    int* b;
}

void main()
{
    immutable A a;   // both a.a and a.b are immutable
}
```

Although this design has a lot of advantages, there are some situations where some fields
might need to be kept mutable, no matter how the instace is qualified. The following
real life examples will illustrate the problem:

1. The usage of the `RefCounted` struct [1] in the `Array` struct [2] in phobos:

```d
struct RefCounted(T)
{
    T data;
    size_t count;

    /* methods of RefCounted */
}

struct Array(T)
{
    private struct Payload
    {
        T[] payload;
        size_t capacity;
    }
    private RefCounted!Payload _data;

    /* methods of Array */
}

void main()
{
    immutable Array!(int*) arr;
}
```

In the above situation, we declare an `immutable` array. By the current language rules,
the reference counted field `_data` is `immutable` which makes the `count` member of
`RefCounted` to also be `immutable`. This results in the imposibility to count the
references for immutable objects in a completely encapsulated manner.

2. The usage of `Rebindable` [3] in `std.datetime` [4]:

```d
struct Rebindable(T)
{
    import std.traits : Unqual;
    private union
    {
        T original;
        Unqual!T stripped;
    }

    /* other, otherwise interesting, methods */
}

class TimeZone {}

struct SysTime
{
    Rebindable!(immutable TimeZone) _timezoneStorage;
}

void main()
{
    immutable SysTime a;
}

```

When a `SysTime` instance is declared as `immutable`, the `Rebindable` object in `SysTime` is also
made `immutable` rendering it useless: the whole point of a rebindable object is to bind it to
something else.

3. Allocators are usually declared as part of the object they are used for. If an instance of the mentioned
object is `immutable`, the allocator will not be able to modify its internal data.

In all of these situations the transitivity of qualifiers makes it difficult to write clean, encapsulated
code. In order to mitigate these issues, a mean of breaking the transitivity of qualifiers is needed. This
is were `__mutable` steps in: it can be used in the declaration of a member field to excerpt it from the
qualifier propagation of a certain instance. For example:

```d
struct RefCounted(T)
{
    T data;
    size_t count;

    /* methods of RefCounted */
}

struct Array(T)
{
    private struct Payload
    {
        T[] payload;
        size_t capacity;
    }
    private __mutable RefCounted!Payload _data;

    /* methods of Array */
}

void main()
{
    immutable Array!(int*) arr;
}
```

Now that the `RefCounted` object is marked as `__mutable`, it can be modified internally, by both
its methods and those of `Array`, and therefore it can be used properly to track the reference count
of that particular object, even though the object itself is `immutable`.

### Necessity of `__mutable` for function declarations

The D language implements a purity system that is based on strongly and weakly `pure` functions [5].
The primary benefit of `pure` functions is that they are building blocks for compiler optimizations,
however, currently, the DMD compiler does not apply any of the optimization techniques that could
be employed. The reason for this is that the weak purity model invalidates the basic optimization
techniques used for strongly `pure` functions when it comes to deallocation. In order to better
understand the nature of the problem, let us see what kind of optimizations may be applied for strongly
`pure` functions and how can these be used for D code.

Some of the basic optimizations for strongly `pure` functions are:
1. A strongly `pure` function whose result is not used can be safely elided.
2. A strongly `pure` function invocation can always exchange order with an adjacent impure function invocation.

The following example highlights the problems that arise when the optimizations are used in conjonction
with functions that deal with deallocation:

```d
void deallocate(immutable T x)
{
    T y = cast(T)x;
    free(y);
}

int foo() pure
{
     immutable(T)* x = allocate();
     int y = bar(x);
     deallocate(x);
     return y;
}
```

`foo` is a strongly `pure` function, since it does not access any global mutable data and it does not
receive any parameters. `allocate`, `bar` and `deallocate` are also strongly `pure` functions. In this
situation the compiler may apply the above stated optimizations, leading to the following possible
transformations:

```d
// optimization 1
int foo() pure
{
     immutable(T)* x = allocate();
     int y = bar(x);
     return y;
} // memory leak

// optimization 2
int foo() pure
{
     immutable T* x = allocate();
     deallocate(x);
     int y = bar(x); // use after free
     return y;
}
```

For the first transformation, bluntly called `optimization 1`, the deallocation of x has been omitted
because `deallocate` is a strongly `pure` function and its result is never used, leading to a memory
leak.

The second transformation is based on the fact that the compiler may choose to swap the two strongly `pure`
function invocations, however, this will lead to an use after free.

In both of the above situations the problem arises because of the bypassing of the language rules by means of
casting, however there is no other way of deallocating `immutable` objects. The solution to this problem is to
signal the compiler that functions that deal with deallocation in a `pure` manner should never be optimized;
`__mutable` can be used for that.

## Implementation

This section highlights the implementation details of the `__mutable` keyword.

### `__mutable` member fields

The `__mutable` storage class modifies propagation of type qualifiers on fields. `__mutable` data can only be manipulated in `@system` code and `__mutable` can only be applied to `private` members.

```d
struct S
{
    int* p;
    shared int* s;
    private __mutable
    {
        int* m;
        shared int* ms;
    }

    private __mutable const int* mc; // error: cannot be both __mutable and const
    private __mutable immutable int* imc; // error: cannot be both __mutable and immutable
    __mutable int* pm; // error: `__mutable` fields must be `private`
}

struct T
{
    private __mutable int* a;
    private
    {
        __mutable int* b;
    }
    __mutable
    {
        private int* c;
    }
}

static assert(!__traits(isUnderUnderMutable, S.p));
static assert(__traits(isUnderUnderMutable, S.m));

__mutable int x = 2; // error: globals and statics cannot be `__mutable`

int* bar() @safe
{
    S s;
    return s.m; // error: cannot access `__mutable` field `m` in `@safe` function `bar`
}

void foo(inout(int)*) @safe
{
    __mutable int* p; // error: __mutable can only be applied to field of aggregate type.
    S s;
    const(S) cs;
    immutable(S) is_;
    shared(S) ss;
    shared(const(S)) css;
    inout(S) ws;
    const(inout(S)) cws;
    shared(inout(S)) sws;
    shared(const(inout(S))) scws;

    // for non-__mutable fields, qualifiers are propagated (existing behavior):
    static assert(is(typeof(s.p)==int*));
    static assert(is(typeof(cs.p)==const(int*)));
    static assert(is(typeof(is_.p)==immutable(int*)));
    static assert(is(typeof(ss.p)==shared(int*)));
    static assert(is(typeof(css.p)==shared(const(int*))));
    static assert(is(typeof(ws.p)==inout(int*)));
    static assert(is(typeof(cws.p)==const(inout(int*))));
    static assert(is(typeof(sws.p)==shared(inout(int*))));
    static assert(is(typeof(scws.p)==shared(const(inout(int*)))));

    static assert(is(typeof(s.s)==shared(int*)));
    static assert(is(typeof(cs.s)==shared(const(int*))));
    static assert(is(typeof(is_.s)==immutable(int*)));
    static assert(is(typeof(ss.s)==shared(int*)));
    static assert(is(typeof(css.s)==shared(const(int*))));
    static assert(is(typeof(ws.s)==shared(inout(int*))));
    static assert(is(typeof(cws.s)==shared(const(inout(int*)))));
    static assert(is(typeof(sws.s)==shared(inout(int*))));
    static assert(is(typeof(scws.s)==shared(const(inout(int*)))));

    // for __mutable fields, qualifier propagation is modified:
    static assert(is(typeof(s.m)==int*));
    static assert(is(typeof(cs.m)==shared(int*)));
    // Requires explicit cast to mutable or shared:
    static assert(is(typeof(*cast(int**)&cs.m))); // explicit reinterpret-casts are allowed
    static assert(is(typeof(*cast(shared(int)**)&cs.m))); // explicit reinterpret-casts are allowed
    static assert(is(typeof(is_.m)==shared(int*))); // immutable is implicitly shared
    static assert(is(typeof(ss.m)==shared(int*)));
    static assert(is(typeof(css.m)==shared(int*)));
    static assert(is(typeof(cs.m)==int*));
    static assert(is(typeof(ws.m)==shared(int*)));
    static assert(is(typeof(*cast(int**)&ws.m))); // explicit reinterpret-cast
    static assert(is(typeof(*cast(shared(int)**)&ws.m))); // explicit reinterpret-cast
    static assert(is(typeof(cws.m)==shared(int*)));
    static assert(is(typeof(*cast(int**)&cws.m))); // explicit reinterpret-cast
    static assert(is(typeof(*cast(shared(int)**)&cws.m))); // explicit reinterpret-cast
    static assert(is(typeof(sws.m)==shared(int*)));
    static assert(is(typeof(scws.m)==shared(int*)));

    static assert(is(typeof(s.ms)==shared(int*)));
    static assert(is(typeof(cs.ms)==shared(int*)));
    static assert(is(typeof(is_.ms)==shared(int*)));
    static assert(is(typeof(ss.ms)==shared(int*)));
    static assert(is(typeof(css.ms)==shared(int*)));
    static assert(is(typeof(ws.ms)==shared(int*)));
    static assert(is(typeof(cws.ms)==shared(int*)));
    static assert(is(typeof(sws.ms)==shared(int*)));
    static assert(is(typeof(scws.ms)==shared(int*)));
}
```

I.e. `immutable` and `const` will be transitive with the exception of `__mutable` fields. Mutating `__mutable` fields has defined behavior. This also holds for the cases where typing rules cannot determine the `shared` status: in `@system` code, casting to the correct `shared`/`unshared` mutable type and mutating the memory is defined behavior.

Mutating memory through typecast `const`/`immutable` references continues to possibly cause undefined behavior.

### `__mutable` functions

`__mutable` functions must be `pure` and `@system`/`@trusted`. `__mutable` function invocations are exempt from purity related optimization techniques.
