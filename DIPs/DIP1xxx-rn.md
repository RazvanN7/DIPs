__mutable storage class
=======================

Proposal
--------

We introduce a new storage class `__mutable` that can be applied to fields of aggregates.

The `__mutable` storage class modifies propagation of type qualifiers on fields. `__mutable` data can only be manipulated in `@system` code and `__mutable` can only be applied to `private` members.

```d
struct S{
	int* p;
	shared int* s;
	private __mutable{
		int* m;
		shared int* ms;
	}
	//private __mutable const int* mc; // error: cannot be both __mutable and const
	//private __mutable immutable int* imc; // error: cannot be both __mutable and immutable
	//__mutable int* pm; // error: `__mutable` fields must be `private`
}

struct T{
	private __mutable int* a;
	private{
		__mutable int* b;
	}
	__mutable{
		private int* c;
	}
}

static assert(!__traits(isUnderUnderMutable, S.p));
static assert(__traits(isUnderUnderMutable, S.m));

//__mutable int x = 2; // error: globals and statics cannot be `__mutable`

/+int* bar()@safe{
	S s;
	return s.m; // error: cannot access `__mutable` field `m` in `@safe` function `bar`
}
+/

void foo(inout(int)*)@safe{
    // __mutable int* p; // error: __mutable can only be applied to field of aggregate type.
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
	static assert(is(typeof(cs.m)==int*)); // TODO: good?
	// Requires explicit cast to mutable or shared:
	static assert(is(typeof(*cast(int**)&cs.m))); // explicit reinterpret-casts are allowed
	static assert(is(typeof(*cast(shared(int)**)&cs.m))); // explicit reinterpret-casts are allowed
	static assert(is(typeof(is_.m)==shared(int*))); // immutable is implicitly shared
	static assert(is(typeof(ss.m)==shared(int*)));
	static assert(is(typeof(css.m)==shared(int*)));
	static assert(is(typeof(cs.m)==int*));
	static assert(is(typeof(ws.m)==int*)); // TODO: good?
	static assert(is(typeof(*cast(int**)&ws.m))); // explicit reinterpret-cast
	static assert(is(typeof(*cast(shared(int)**)&ws.m))); // explicit reinterpret-cast
	static assert(is(typeof(cws.m)==int*)); // TODO: good?
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

I.e. `immutable` and `const` will be transitive with the exception of `__mutable` fields. Mutating `__mutable` fields has defined behavior.
This also holds for the cases where typing rules cannot determine the `shared` status: in `@system` code, casting to the correct `shared`/un`shared` mutable type and mutating the memory is defined behavior.

Mutating memory through typecast `const`/`immutable` references continues to possibly cause undefined behavior.

Interaction with `pure`
=======================

We want strong guarantees even with `__mutable`, therefore we should define `pure` and `immutable` in terms of transformations/rules. To define the rules, I will use function invocations for simplicity, but the same rules would apply to sequences of statements. Imagine the D program in SSA, and consider any assignment right hand side a "function", either pure or impure.


(*NOTE": The following idea might need some more work.

Let "basic semantics" be the semantics of the D programming language when eliding all type qualifiers.

The sense in which those rules hold is:

- A program that does not throw an error nor has undefined behavior, but can be transformed into a program that throws an error or has undefined behavior, itself has undefined behavior.

- A program that throws an error using basic semantics, but can be transformed into a program that does not throw an error using basic semantics has undefined behavior.
)

Basic Rules
-----------

- A strongly pure function (that return types without indirections) will return the same result when applied to the same `immutable` arguments. (Regardless of how many intermediate interactions happen.) For return types with indirections, the function will always return a data graph with isomorphic mutable aliasing.

- The set of references returned from strongly `pure` functions can be safely converted to `immutable` or `shared`.

- A strongly `pure` function whose result is not used may be safely elided.

- If we have two subsequent `pure` function invocations `foo(args1...)` and `bar(args2...)` where data transitively reachable from `args1` and `args2` only overlaps in `immutable` and `const` data (this includes data accessed through `__mutable` fields of `const` and `immutable` objects), the two invocations may safely swap their order (of course, this only applies if none of the two functions takes arguments).

- A strongly `pure` function invocation can always exchange order with an adjacent impure function invocation.


I.e., the two programs

```
// create args1/args2, and more arguments
auto a = foo(args1...);
auto b = bar(args2...);
// use a and b
```

and

```
// create args1/args2, and more arguments
auto b = bar(args2...);
auto a = foo(args1...);
// use a and b
```

are always interchangeable.

__mutable functions
-------------------

Functions that are annotated with `__mutable` are exempt from those rules. `__mutable` functions must be `pure` and `@system`/`@trusted`. `@trusted` `pure` functions that call `@system` `__mutable` functions therefore must ensure that the above rules will be valid for `pure` functions calling the `@trusted` functions.

*IMPORTANT*: Note that this means that accessing memory addresses and identities of `immutable` data must be made un`@safe` in `pure` functions. (For example, with `is` expressions.)

Rationale: `__mutable` functions can be used to conveniently break the rules while implementing abstractions, to finally build up a higher-level abstraction that satisfies the rules. One use case for `__mutable` functions is deallocation.


Common Subexpression Elimination
--------------------------------

The following rule enables common subexpression elimination:

- `__mutable` functions can only be called from other `__mutable` functions or `pure` destructors.

(Note that this rule need not be part of the language definition, as there may be more loose conditions that make the rule below work, but it would make sense to make the type system check it.)

We can then add the following common subexpression elimination rule:

- If we have two (identical) strongly `pure` (non-`__mutable`) function calls `auto a = foo(arg)` and `auto b = foo(arg)` we can transform to `auto a = foo(arg)` and `auto b = a`.

Rationale: This allows common subexpression elimination/referential transparency, the main important property of `pure` functions.
           The main issue with common subexpression elimination is that deallocation may become tricky unless it all happens in destructors.


Other Considerations
====================

- It is a small problem that `immutable` is implicitly `shared`. This only makes total sense when `immutable` memory cannot be modified transitively.
  `__mutable` breaks this, therefore there is no safe way to support `__mutable` for `const` references. The solution proposed above essentially requires the implementation to record whether the current object is shared or not.

  Alternatively, we might consider having separate (unshared) `immutable` and `shared immutable` types, such that we know that `const` memory is unshared and `shared const` memory is shared.

