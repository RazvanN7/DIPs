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

But there are situations where specific fields might need to be kept mutable,
no matter how the instance is qualified. The following real life examples
will illustrate the problem:

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

## Semantics

The `__mutable` storage class modifies propagation of type qualifiers on fields. `__mutable` can only be applied to `private`, mutable members.

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
    __mutable int* pm; // error: `__mutable` fields must be `private`
    private __mutable const int* mc; // error: cannot be both __mutable and const
    private __mutable immutable int* imc; // error: cannot be both __mutable and immutable
}
```

`immutable` and `const` will be transitive with the exception of `__mutable` fields. Mutating `__mutable` fields has defined behavior:

```d
struct S
{
    int p;
    private __mutable int m;

    void increment() immutable
    {
        ++m;       // ok, m is __mutable
        ++p;       // error: cannot modify field p of immutable instance of S
    }
}
```

`__mutable` does not apply transitively:

```d
struct RefCount
{
    immutable size_t id;
    size_t count;
}

struct T
{
    private __mutable RefCount s;

    void updateRC() immutable
    {
        ++s.count;    // fine
        ++s.id;       // error: cannot modify immutable field id
    }
}
```

Global/Local and static variables cannot be `__mutable`:

```d
module jaja;
__mutable int x = 2; // error: global variable cannot be `__mutable`
int fun()
{
    __mutable int y;    // error: local variable cannot be `__mutable`
}

struct S
{
    private `__mutable` static int b;    // error: static variable cannot be `__mutable`
}
```

`__mutable` data can only be manipulated in `@system` code:

```d
struct S
{
    private __mutable int* m;
}

int* bar() @safe
{
    S s;
    return s.m; // error: cannot access `__mutable` field `m` in `@safe` function `bar`
}
```

Because `immutable` is implicitly `shared`, `__mutable` fields of `immutable` instances
are regarded as `shared` in order to be thread-safe:

```d
struct S
{
    private `__mutable` int* p;
}
immutable S s;
static assert(is(typeof(s.m) == shared(int*)));
```

`const` references to objects may come from mutable, `const` and `immutable` objects, therefore
the type system cannot infer whether the `__mutable` field inside the object should be shared
or unshared. `__mutable` fields of `const` objects will be type checked as being unshared mutable,
leaving upon the user to reason upon the code make the appropriate casts:

```d
shared int* q;
int* r;

struct A
{
    private __mutable int* p;
    bool isImmutable;

    this(shared int *p) immutable
    {
        this.p = p;
        isImmutable = true;
    }

    void fun() const
    {
        pragma(msg, typeof(p).stringof); // typeof(p) => int*
        if (isImmutable)
            q = cast(shared int*) p;
        else
            r = p;
    }
}

void main()
{
    shared int* p
    immutable A ia = immutable A(p);
    const A ca;
    ia.fun();
    ca.fun();
}
```

`inout` references to objects may come from mutable, `const` and `immutable` objects. This similarity to
`const` objects makes `inout` object instances with `__mutable fields` to be treated exactly the same as
`const` ones.

With the addition of `__mutable`, references to `immutable` instances do not offer the same purity guarantees when passed as function arguments:

```d
struct S
{
    private `__mutable` int* p;

    void inc() immutable pure
    {
        *p += 1;
    }
}

void foo(ref immutable S a) pure
{
    a.inc();
}

int x;

void main()
{
    A b = A(&x);
    foo(b);
}
```

Before this DIP, `foo` would have been regarded as a strongly pure function, but with the addition
of `__mutable`, `foo` becomes a weakly `pure` function due to the fact that `immutable` instances
can now be modified through `__mutable fields`. After this DIP, a function that receives immutable
ref/ptr parameters will be regarded as strongly `pure` if and only if the parameters do not contain
any `__mutable` fields.
