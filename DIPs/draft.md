# Perfect Forwarding

## Current situation:

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


## @forward

void wrapper(T)(@forward auto ref T a) { callee(a); }

1. `@forward` can be used solely with `auto ref` parameters.
2. `@forward` does not affect overload resolution.
3. After overload resolution and template instantiation a `@forward` parameter can be either `@forward T` or `@forward ref T`, but the compiler always passes the argument by reference.
4. Inside the function body `@forward T` means "a reference to an rvalue", `@forward ref T` means "a reference to an lvalue".
5. When calling a function with an `@forward T` argument we have the following situations:

   * the callee receives the argument by `ref`; in this situation the compiler will issue an error: "Cannot take a reference to an rvalue"
   * the callee receives the argument by value; in this situation a move is performed
   * the callee receives the argument by `@forward auto ref`; in this situation the callee will be instantianted to receive `@forward` i.e. a reference to an rvalue.

6. When calling a function with an `@forward ref T` argument, the rules are the same as if it was a simple `ref T` argument.

7. When a code is compiled with `-preview=rvaluerefparam`, the semantics do not change because `auto ref` will still bind the rvalue
to the non-ref function (otherwise `auto ref` can simply be translated to `ref`).
8. If the last access rules are implemented, `@forward T` parameters are exempt from those rules.
9. `@forward T` parameters (or rvalue references) cannot be passed to `ref` parameters, except if the parameter is marked as `@forward`.
10. `@forward T ` parameters act as rvalues when they are used as function arguments, but they act as lvalues in all other situations. If a `@forward` variable is used after it was moved an error is issued.
11. Member variables of `@forward` parameters are treated under the same rules as non-`@forward` parameters.
The purpose of `@forward` is to instruct the compiler that a specific parameter is going to be forwarded to
another function at some point. Whatever actions are taken regarding the fields of an `@forward` object must
not alter it before it was forwarded.
12. It is unsafe to take the address of a `@forward T` parameter.

Example:

1. void callee(S a):

  * wrapper(a)    => 0 (@forward ref S) + 1 copy = 1 copy
  * wrapper(S())  => 0 (@forward S)     + 1 move (rule 5b) = 1 move

2. void callee(ref S a):

  * wrapper(a)   => 0 (@forward ref S) + 0 (by ref) = 0
  * wrapper(S()) => 0 (@forward S)     + error(rule 5a)    = error



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
o
