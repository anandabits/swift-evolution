# Parameter Forwarding

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-parameter-forwarding.md)
* Author(s): [Matthew Johnson](https://github.com/anandabits)
* Status: **Awaiting review**
* Review manager: TBD

## Introduction

This feature introduces an automatic parameter forwarding mechanism.

Swift-evolution thread: [Proposal Draft: Parameter Forwarding](https://lists.swift.org/pipermail/swift-evolution)

## Motivation

There are many cases where a function declares parameters simply for the purpose of forwarding the provided arguments to another function.  This results in reduntant parameter specifications that make code less clear and less readable by obscuring the simple forwarding that is actually happening.

This feature will be especially useful in initializers such as:

* Convenience initializers that foward parameters directly to a designated initializer
* Designated initializers that foward parameters directly to a super initializer
* Designated initializers that foward parameters directly to a member initializer, perhaps in a composition-based design
* If the partial initilaizer proposal is accepted, designated initializers that forward parameters to one or more partial initializers

NOTE: I haven't had time to think too much aboue use cases beyond initialization.  Please share examples and I will add them to this proposal.

## Proposed solution

The proposed solution is to introduce an automatic parameter forwarding mechansim.  It allows users to provide direct arguments for some parameters while forwarding others.

The basic mechanism looks like this:

```swift
func foo(i i: Int, s: String, f: Float = 42, d: Double = 43, b: Bool = false) { }

// user writes:
func bar(...fooParams) {
	foo(i: 32, ...fooParams)
}

// compiler synthesizes:
func bar(s: String, f: Float = 42, d: Double = 43, b: Bool = false) {
	foo(i: 32, s: s, f: f, d: d, b: b)
}
```

Some things to note about the syntax:

1. `...fooParams` is a placeholder introduced with `...` and followed by an identifier.  
2. In the signature it can be placed anywhere in the parameter list.  
3. At the call site, it must appear at the end of the argument list.  
4. The placeholder matches the parameters not directly provided including their external label and default value if those exist.  
5. Parameters corresponding to the matched parameters are synthesized by the compiler where the placeholder exists in the parameter list, including the default argument if one exists.  
6. The identifier portion of the placeholder may be omitted if only one set of forwarded parameters exist within the function.

Additional details will be introduced with a corresponding example.

### Omitting the placeholder identifier

The above example can be written more concisely by omitting the placeholder identifier.

```swift
func foo(i i: Int, s: String, f: Float = 42, d: Double = 43, b: Bool = false) { }

// user writes:
func bar(...) {
	foo(i: 32, ...)
}

// compiler synthesizes:
func bar(s: String, f: Float = 42, d: Double = 43, b: Bool = false) {
	foo(i: 32, s: s, f: f, d: d, b: b)
}
```

NOTE: If the community feels strongly that the identifier should be required I am willing to do so.

### Multiple forwarded parameter sets

It is possible for a single function to forward more than one set of parameters:

```swift
func foo(i i: Int, s: String, f: Float = 42) { }
func foo2(d: Double = 43, b: Bool = false) { }

// user writes:
func bar(...fooParams, ...foo2Params) {
	foo2(...foo2Params)
	foo(i: 32, ...fooParams)
}

// compiler synthesizes:
func bar(s: String, f: Float = 42, d: Double = 43, b: Bool = false) {
	foo(i: 32, s: s, f: f, d: d, b: b)
}
```

### Direct arguments

Any direct arguments provided in the forwarding call must follow the usual argument ordering rules, with the only exception being that it is allowed to omit some arguments that would normally be required.  When the compiler performs forwarding it will insert forwarded arguments in the correct location.

```swift
func foo(i i: Int, s: String, f: Float = 42, d: Double = 43, b: Bool = false) { }

func bar(...fooParams) {
	// error: `i` must precede `s` in the argument list
	foo(s: "hello", i: 32, ...fooParams)
}

// user writes:
func bar(...fooParams) {
	foo(i: 32, f: 0, ...fooParams)
}

// compiler synthesizes:
func bar(s s: String, d: Double = 43, b: Bool = false) {
	foo(i: 32, s: s, f: 0, d: d, b: b)
}
```

### Multi-forwarding the same parameters

It is allowed to use the same identifier in multiple forwarding calls as long as the signature of the matched parameters matches exactly, including any default values.  

```swift
func foo(i i: Int, s: String, d: Double = 43) { }
func bar(i i: Int, s: String, d: Double = 43) { }

// user writes:
func baz(...fooBarParams) {
	foo(...fooBarParams)
	bar(...fooBarParams)
}

// compiler synthesizes: 
func baz(i i: Int, s: String, d: Double = 43) {
	foo(i: i, s: s, d: d)
	bar(i: i, s: s, d: d)
}
```

NOTE: This provision might be controversial.  If the community doesn't like it or the implementation is too complex I will remove it.


### Unambiguous call

When forwarding parameters to a function that is overloaded the caller must provide enough direct arguments to make the call unambiguous.

```swift
func foo(i i: Int, s: String, d: Double = 43, b: Bool = false) { }
func foo(i i: Int, s: String, d: Double = 43, f: Float = 42) { }

// user writes:
func bar(...fooParams) {
	// error: ambiguous use of foo
	// foo(i: 32, ...fooParams)
	
	// ok: `b` makes the call to foo unambiguous
	foo(b: true, ...fooParams)
	// ok: `f` makes the call to foo unambiguous
	foo(f: 24, ...fooParams)
}

// compiler synthesizes: 
func bar(i i: Int, s: String, d: Double = 43) {
	foo(i: i, s: s, d: d, b: true)
	foo(i: i, s: s, d: d, f: 24)
}
```

### Default values

When forwarding to a function that accepts default values it is possible to explicitly request the default value.  This allows for disambiguation and also allows the forwarding function to suppress a defaulted parameter from participating in forwarding without needing to supply a specific value.  The `default` keyword is used to do this.

We can modify the previous example to use the defualt values:

```swift
func foo(i i: Int, s: String, d: Double = 43, b: Bool = false) { }
func foo(i i: Int, s: String, d: Double = 43, f: Float = 42) { }

// user writes:
func bar(...fooParams) {
	// ok: `b` makes the call to foo unambiguous, still uses default value
	foo(b: default, ...fooParams)
	// ok: `f` makes the call to foo unambiguous, still uses default value
	foo(f: default, ...fooParams)
}

// compiler synthesizes:
func bar(i i: Int, s: String, d: Double = 43) {
	foo(i: i, s: s, d: d, b: false)
	foo(i: i, s: s, d: d, f: 42)
}
```

It is also possible to explicitly request all defaults at once using `default...`.  In this example, `foo` is not overloaded:

```swift
func foo(i i: Int, s: String, d: Double = 43, b: Bool = false) { }

// user writes:
func bar(...fooParams) {
	foo(default..., ...fooParams)
}

// compiler synthesizes:
func bar(i i: Int, s: String) {
	foo(i: i, s: s, d: 43, b: false)
}
```

NOTE: The actual implementation of default arguments looks somewhat different.  These examples are intended to communicate the behavior, not the exact details of implementation.

### Generic parameters

If the types of any matched parameters reference any generic type parameters of the forwardee the generic type parameters must also be forwarded, along with any constraints on those generic parameters. 

```swift
func foo<T>(i i: Int, s: String, t: T, d: Double = 43, b: Bool = false) { }

// user writes:
func bar(...fooParams) {
	foo(...fooParams)
}

// compiler synthesizes:
func bar<T>(i i: Int, s: String, t: T, d: Double = 43, b: Bool = false) {
	foo(i: i, s: s, t: t, d: d, b: b)
}
```

If a generic parameter is referenced in a constraint that also references a generic parameter that will not be forwarded the constraint is resolved to a concrete type when possible.  This may not be possible in all cases.  When it is not possible a compiler error will be necessary.

```swift
func foo<S: SequenceType, T: SequenceType where S.Generator.Element == T.Generator.Element>
	(s: S, t: T) { }

// user writes:
func bar(...fooParams) {
	foo(t: [42], ...fooParams)
}

// compiler synthesizes:
func bar<S: SequenceType where S.Generator.Element == Int>(s: S) {
	foo(s: s, t: [42])
}
```

### Syntheszied internal names

The compiler must ensure that all synthesized parameters have internal names that do not conflict with the internal names of any manually declared parameters.  This applies to both generic type parameter names as well as value arguments in the parameter list of the function.

```swift
func foo<T>(i i: Int, s: String, t: T, d: Double = 43, b: Bool = false) { }

// user writes:
func bar<T>(t: T, ...fooParams) {
	// do something with t
	foo(...fooParams)
}

// compiler synthesizes:
func bar<T, InternalCompilerIdentifier>(t: T, i i: Int, s: String, t internalCompilerIdentifier: InternalCompilerIdentifier, d: Double = 43, b: Bool = false) {
	foo(t: t, i: i, s: s, t: internalCompilerIdentifier, d: d, b: b)
}
```

## Detailed design

TODO but should fall out pretty clearly from the proposed solution

## Impact on existing code

This is a strictly additive change.  It has no impact on existing code.

## Alternatives considered

I believe the forwarding mechanism itself is pretty straightforward and any alternatives would be lose functionality without good reason.  

The placeholder syntax is of course fair game for bikeshedding.  I consider anything reasonably clear and concise to be acceptable.
