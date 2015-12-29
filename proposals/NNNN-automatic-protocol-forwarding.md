# Automatic protocol forwarding

* Proposal: [SE-NNNN](https://github.com/anandabits/swift-evolution/blob/automatic-protocol-forwarding/proposals/NNNN-automatic-protocol-forwarding.md)
* Author(s): [Matthew Johnson](https://github.com/anandabits)
* Status: **Review**
* Review manager: TBD

## Introduction

Automatic protocol forwarding introduces the ability to use delegation without the need write forwarding member implementations manually. 

A preliminary mailing list thread on this topic had the subject [protocol based invocation forwarding](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151207/000931.html)

Swift-evolution thread: [Proposal Draft: automatic protocol forwarding](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151228/004737.html)

## Motivation

Delegation is a robust, composition oriented design technique that keeps interface and implementation inheritance separate.  The primary drawback to this technique is that it requires a lot of manual boilerplate to forward implemenation to the implementing member.  This proposal eliminates the need to write such boilerplate manually, thus making delegation-based designs much more convenient and attractive.

This proposal may also serve as the foundation for a future enhancement allowing a very concise "newtype" declaration.

## Proposed solution

I propose introducing a `forward` declaration be allowed within a type declaration or type extension.  The `forward` declaration will cause the compiler to synthesize implementations of the members required by the forwarded protocols.  The synthesized implementations will simply forward the method call to the specified member.

The basic declaration looks like this:

```swift
forward Protocol, OtherProtocol to memberIdentifier
```

The first clause contains a list of protocols to forward.  

The second clause specifies the identifier of the property to which the protocol members will be forwarded.  Any visible property that implements the members required by the protocol is eligible for forwarding.  It does not matter whether it is stored, computed, lazy, etc.

It is also possible to include an access control declaration modifier to specify the visibility of the synthesized members.

### Self parameters

When a protocol member includes a `Self` parameter forwarding implementations must accept the **forwarding** type but supply an argument of the **forwardee** type when making the forwarding call.  The most straightforward way to do this is to simply use the same property getter that is used when forwarding.  This is the proposed solution.

### Self return types

When a protocol member includes a `Self` return type forwarding implementations must return the **forwarding** type.  However, the **forwardee** implmentation will return a value of the **forwardee** type.  This result must be used to produce a value of the **forwarding** type in some way.

The solution in this proposal is based on an ad-hoc overloading convention.  A protocol-based solution would probably be desirable if it were possible, however it is not.  This proposal supports forwarding to more than one member, possibly with different types.  A protocol-based solution would require the *forwarding* type to conform to the "`Self` return value conversion" protocol once for each forwardee type.

#### Static members

When a **forwardee** value is returned from a static member an initializer will be used to produce a final return value.  The initializer must be visible at the source location of the `forward` declaration and must look like this:


```swift
struct Forwarder {
	let forwardee: Forwardee
	forward P to forwardee
	init(_ forwardeeReturnValue: Forwardee) { //... }
}

```

#### Instance members

When a **forwardee** value is returned from an instance member an instance method will be used to transform the return value into a value of the correct type.  An instance method is necessary in order to allow the *forwarding* type to access the state of the instance upon which the method was called when performing the transformation.

If the instance method is not implemented the initializer used for static members will be used instead.

The transformation has the form:

```swift
struct Forwarder {
	let forwardee: Forwardee
	forward P to forwardee
	func transformedForwardingReturnValue(forwardeeReturnValue: Forwardee) -> Forwarder { //... }
}

```

NOTE: This method should have a better name.  Suggestions are appreciated!

### Examples

#### Basic example

NOTE: `Forwardee` does not actually conform to `P` itself.  Conformance is not required to synthesize the forwarding member implementations.  It is only required that members necessary for forwarding exist.  This is particularly important to the second example.

```swift
public protocol P {
	typealias TA
	var i: Int
	func foo() -> Bool
}

private struct Forwardee {
	typealias TA = String
	var i: Int = 42
	func foo() -> Bool { return true }
}

public struct Forwarder {
	private let forwardee: Forwardee
}

extension Forwarder: P {	
	// user declares
	public forward P to forwardee
	
	// compiler synthesizes
	// TA must be synthesized as it cannot be inferred for this protocol
	public typealias TA = String 
	public var i: Int {
		get { return forwardee.i }
		set { forwardee.i = newValue }
	}
	public func foo() -> Bool { 
		return forwardee.foo() 
	}
}
```

#### Existential forwardee

NOTE: Existentials of type `P` do not actually conform to `P` itself.  Conformance is not required to synthesize the forwarding member implementations.  It is only required that members necessary for forwarding exist.

```swift
public protocol P {
	func foo() -> Bool
}

struct S: P {
	private let p: P
	
	// user declares:
	forward P to p
	
	// compiler synthesizes:
	func foo() -> Bool {
		return p.foo()
	}
}
```

#### Self parameters

```swift
public protocol P {
	func foo(value: Self) -> Bool
}

extension Int: P {
	func foo(value: Int) -> Bool {
		return value != self
	}
}

struct S: P {
	private let i: Int
	
	// user declares:
	forward P to i
	
	// compiler synthesizes:
	func foo(value: S) -> Bool {
		return i.foo(value.i)
	}
}
```

#### Self return types

Using the instance method:

```swift
public protocol P {
	func foo() -> Self
}

extension Int: P {
	func foo() -> Int {
		return self + 1
	}
}

struct S: P {
	private let i: Int
	func transformedForwardingReturnValue(forwardeeReturnValue: Int) -> S {
		return S(i: forwardeeReturnValue)
	}
	
	// user declares:
	forward P to i
	
	// compiler synthesizes:
	func foo() -> S {
		return self.transformedForwardingReturnValue(i.foo())
	}
}
```

Using the initializer:

```swift
public protocol P {
	func foo() -> Self
}

extension Int: P {
	func foo() -> Int {
		return self + 1
	}
}

struct S: P {
	private let i: Int
	init(_ value: Int) {
		i = value
	}
	
	// user declares:
	forward P to i
	
	// compiler synthesizes:
	func foo() -> S {
		return S(i.foo())
	}
}
```

#### Forwarding multiple protocols


```swift
public protocol P {
	func foo() -> Bool
}
public protocol Q {
	func bar() -> Bool
}

extension Int: P, Q {
	func foo() -> Bool {
		return true
	}
	func bar() -> Bool {
		return false
	}
}

struct S: P, Q {
	private let i: Int
	
	// user declares:
	forward P, Q to i
	
	// compiler synthesizes:
	func foo() -> Bool {
		return i.foo()
	}
	func bar() -> Bool {
		return i.bar()
	}
}
```

#### Forwarding to multiple members

```swift
public protocol P {
	func foo() -> Bool
}
public protocol Q {
	func bar() -> Bool
}

extension Int: P {
	func foo() -> Bool {
		return true
	}
}
extension Double: Q {
	func bar() -> Bool {
		return false
	}
}

struct S: P, Q {
	private let i: Int
	private let d: Double
	
	// user declares:
	forward P to i
	forward Q to d
	
	// compiler synthesizes:
	func foo() -> Bool {
		return i.foo()
	}
	func bar() -> Bool {
		return d.bar()
	}
}
```

#### Non-final class

NOTE: `C` cannot declare conformance to the protocol due to the `Self` return value requirement.  However, the compiler still synthesizes the forwarding methods and allows them to be used directly by users of `C`.

```swift
public protocol P {
	func foo() -> Self
}

extension Int: P {
	func foo() -> Int {
		return self + 1
	}
}

// C does not and cannot declare conformance to P
class C {
	private let i: Int
	init(_ value: Int) {
		i = value
	}
	
	// user declares:
	forward P to i
	
	// compiler synthesizes:
	func foo() -> C {
		return C(i.foo())
	}
}
```


## Detailed design

TODO: grammar modification to add the `forward` declaration

1. Automatic forwarding only synthesizes member implementations.  It does not automatically conform the forwarding type to the protocol(s) that are forwarded.  If actual conformance is desired (as it usually will be) it must be explicitly stated.

2. The forwardee type need not actually conform to the protocol forwarded to it.  It only needs to implement the members the forwarder must access in the synthesized forwarding methods.  This is particularly important as long as protocol existentials do not conform to the protocol itself.

3. While it will not be possible to conform non-final classes to protocols containing a `Self` return type forwarding should still be allowed.  The synthesized methods will have a return type of the non-final class which in which the forwarding declaration occured.  The synthesized methods may still be useful in cases where actual protocol conformance is not necessary.

3. All synthesized members recieve access control modifiers matching the access control modifier applied to the `forward` declaration.

4. TODO: How should other annotations on the forwardee implementations of forwarded members (such as @warn_unused_result) be handled?  

5. It is possible that the member implementations synthesized by forwarding will conflict with existing members or with each other (when forwarding more than one protocol).  All such conflicts, with one exception, should produce a compiler error at the site of the forwarding declaration which resulted in conflicting members.  
	One specific case that **should not** be considered a conflict is when forwarding more than one protocol with **identical** member declarations to the **same** member of the forwarding type.  In this case the synthesized implementation required to forward all of the protocols is identical.  The compiler **should not** synthesize multiple copies of the implementation and then report a redeclaration error.

6. It is likely that any attempt to forward different protocols with `Self` return types to more than one member of the same type will result in sensible behavior.  This should probable be a compiler error.  For example:

```swift
protocol P {
    func foo() -> Self
}
protocol Q {
    func bar() -> Self
}

struct Forwarder: P, Q {
	let d1: Double
	let d2: Double
	
	forward P to d1
	forward Q to d2

	func transformedForwardingReturnValue(_ forwardeeReturnValue: Double) -> Forwarder { 
		// What do we do here?  
		// We don't know if the return value resulted from forwarding foo to d1 or bar to d2.
		// It is unlikely that the same behavior is correct in both cases.
	}
}
```

## Impact on existing code

This is a strictly additive change.  It has no impact on existing code.

## Future enhancements

In the spirit of incremental change, this proposal focuses on core functionality.  Several enhancements to the core functionality are possible and are likely to be explored in the future.

### Partial forwarding synthesis

The current proposal makes automatic forwarding an "all or nothing" feature.  In cases where you want to forward most of the implementation of a set of members but would need to "override" one or more specific members the current proposal will not help.  You will still be required to forward the entire protocol manually.  Attempting to implement some specific members manually will result in a redeclaration error.

This proposal does not allow partial forwarding synthesis in order to focus on the basic forwarding mechanism and allow us to gain some experience with that first, before considering the best way to make partial forwarding possible without introducing unintended potential for error.  One example of a consideration that may apply is whether or not *forwardee* types should be able to mark members as "final for forwarding" in some way that prevents them from being "overriden" by a forwarder.

### Newtype

While the current proposal provides the basic behavior desired for `newtype`, it is not as concise as it could be.  Adding syntactic sugar to make this common case more concise would be straightforward:

```swift
// user declares
newtype Weight = Double forwarding P, Q

// compiler synthesizes
struct Weight: P, Q {
	var value: Double
	forward P, Q to value
	init(_ value: Double) { self.value = value }
}
```

However, there are [additional nuances related to associated types](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151228/004735.html) that should be considered and addressed by a `newtype` proposal.

### Forwarding declaration in protocol extensions

It may be possible to allow the `forward` declaration in protocol extensions by forwarding to a required property of the protocol.  This may have implementation complexities and other implications which would hold back the current proposal if it required the `forward` declaration to be allowed in protocol extensions.  

## Alternatives considered

### Specify forwarding as part of the member declaration

I originally thought it would make the most sense to specify forwarding alongside the forwardee member declaration.  This proposal does not do so for the following reasons:

1. We must be able to specify access control for the forwarded members that are synthesized.  Introducing a forwarding declaration is the most clear way to allow this.

2. It will sometimes be necessary to forward different protocols to the same forwardee with different access control levels.  It would be very clunky to do this as part of the member declaration.

3. It should be possible to synthesize forwarding retroactively as part of an extension.  This would not be possible if forwarding had to be specified in the original member declaration.

### Require the forwardee to conform to the protocol(s) forwarded to it

There is not a compelling reason to require this.  It is not necessary to synthesize and compile the forwarding methods and it would prevent the use of protocol existentials as the forwardee.

### Automatically conform the forwarding type to the forwarded protocol(s)

It may seem reasonable to automatically synthesize conformance to the protocol in addition to the member implementations.  This proposal does not do so for the following reasons:

1. Forwarding is considered an implementation detail that is not necessarily visible in the public interface of the type.  The forwardee may be a private member of the type.

2. Type authors may wish to control where the actual conformance is declared, especially if protocol conformances are allowed to have access control in the future.

3. There *may* be use cases where it is desirable to have the forwarded members synthesized *without* actually conforming to the protocol.  This is somewhat speculative, but there is not a compelling reason to disallow it.

### Allow forwarding of all protocols conformed to by the forwardee without explicitly listing them

It may seem reasonable to have a `*` placeholder which will forward all visible protocol conformances of the forwardee type.  This proposal does not include such a placeholder for the following reasons:

1. A placeholder like this could lead to unintended operations being synthesized if additional conformances are declared in the future.  The new conformances could even lead to conflicts during synthesis which cause the code to fail to compile.  The potential for such breakage is not acceptable.

2. A placeholder like this would not necessarily cause all desired forwarding methods to be synthesized.  This would be the case when the members necessary to conform exist but actual conformance does not exist.  This would be the case when the forwardee type is an existential.  This could lead to programmer confusion.

3. An explicit list of protocols to forward is not unduely burdensome.  It is straightforward to declare a new protocol that inherits from a group of protocols which are commonly forwarded together and use the new protocol in the forwarding declaration.

4. This is easily added as a future enhancement to the current proposal if we later decide it is necessary.

### Allow forwarding of the entire interface of the forwardee type, not just specific protocols

It is impossible to synthesize forwarding of methods which contain the forwardee type as a parameter or return type that are not declared as part of a protocol interface in a correct and safe manner.  This is because it may or may not be correct to promote the **forwardee** type in the signature to the **forwarder**.  

As an example, consider the following extension to `Double`.  Imagine trying to synthesize a forwarding method in a `Pixel` type that forwards to `Double`.  Should the return type be `Pixel` or `Double`?  It is impossible to tell for sure.

```swift
extension Double {
    func foo() -> Double {
        return self
    }
}
```

When the method is declared in a protocol it becomes obvious what the signature of the forwarding method must be.  If the protocol declares the return type as `Self`, the forwarding method must have a return type of `Pixel`.  If the protocol declares the return type as `Double` the forwarding method will continue to have a return type of `Double`.

### Allow forwarding to `Optional` members

It may seem like a good idea to allow synthesized forwarding to `Optional` members where a no-op results when the `Optional` is `nil`.  There is no way to make this work in general as it would be impossible to forward any member requiring a return value.  If use cases for forwarding to `Optional` members emerege that are restricted to protocols with no members requiring return values the automatic protocol forwarding feature could be enhanced in the future to support these use cases.

### Allow forwarding the same protocol(s) to more than one member

As with forwarding to `Optional` members, forwarding the same protocol to more than one member is not possible in general.  However it *is* possible in cases where no protocol members have a return value.  If compelling use cases emerge to motivate automatic forwarding of such protocols to more than one member an enhancement could be proposed in the future.

### Provide a mechanism for forwardees to know about the forwarder

Some types may be designed to be used as components that are always forwarded to by other types.  Such types may wish to be able to communicate with the forwarding type in some way.  This can be accomplished manually.  

If general patterns emerge in practice it may be possible to add support for them to the language.  However, it would be preliminary to consider support for such a feature until we have significant experience with the basic forwarding mechanism itself.
