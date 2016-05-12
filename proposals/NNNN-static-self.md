# Introducing StaticSelf, an Invariant Self

* Proposal: TBD
* Authors: [Matthew Johnson](https://github.com/anandabits), [Erica Sadun](https://github.com/erica)
* Status: TBD
* Review manager: TBD

## Introduction

This proposal introduces a new keyword that provides consistent invariant type semantics in all contexts.

*The Swift-evolution thread about this topic can be found here: [\[RFC\] #Self](http://thread.gmane.org/gmane.comp.lang.swift.evolution/16565)*

## Motivation

The distinction between covariant and non-covariant type references come into play when
conforming non-final classes to protocols. Fixing a protocol requirement to a covarying type
means that a method returning `Self` must be overriden by all subclasses in order to return
the correct, matching type.

This proposal builds on the covariant construct `Self` accepted in [SE-0068](https://github.com/apple/swift-evolution/blob/master/proposals/0068-universal-self.md)
to introduce an invariant type identifier. It enables protocol declarations to consistently 
refer to a type that is fixed at compile time, ensuring that subclasses can inherit 
protocol implementations without having to re-implement that code at each level of
inheritance.

Under this proposal, a new identifier keyword is fixed in use *at the point of protocol conformance* 
to the static type of that construct. 

```
class A: MyProtocol
```

The invariant identifier will always refer to `A`, unlike `Self`, which is covarying and refers to
the type of the actual instance. Since multiple inheritance for non-protocol types is disallowed, 
this establishes this invariant type identifier with no possibility for conflict.

Consider the following example, under the current system:

```swift
protocol StringCreatable {
    static func createWithString(s: String) -> Self
}

extension NSURL: StringCreatable {
 // cannot conform because NSURL is non-final
 // error: method 'createWithString' in non-final class 'NSURL' must return `Self` to conform to protocol 'A'
}
```

Introducing a static version of `Self` that does not covary would permit the desired conformance:

```swift
protocol StringCreatable {
    static func createWithString(s: String) -> StaticSelf
}

extension NSURL: StringCreatable {
 // can now conform conform because NSURL is fixed and matches the static
 // type of the conforming construct. Subclasses need not re-implement
 // NOTE: the return type can be declared as StaticSelf *or* as NSURL
 //       they are interchangeable
 static func createWithString(s: String) -> StaticSelf { 
     // ...
 }
}
```

### Additional Utility

A secondary use for StaticSelf is to refer to the type of the lexical current context without explicitly mentioning its name.  This may be useful as a shortcut when reference static members of types with very long names.  It may also be useful when code might be copied and pasted.


```swift
class StructWithAVeryLongName {
    static func foo() -> String {
      // ...
    }
    func bar() {
      // ...
      let s = StaticSelf.foo()
      //
    }
}
```

## Detailed Design

This proposal introduces a new keyword `StaticSelf` that may be used in protocols to refer to the invariant static type of a conforming construct.  It may also be used in any lexical context which is enclosed by a type declaration, in which case it is identical to spelling out the full name of that type explicitly.

## Impact on existing code

Being additive, there should be no impact on existing code.

## Alternatives considered

The keyword is not fixed at this time.  Alternatives that have been discussed include `StaticType`, `InvariantSelf`, `SelfType`, or `Type`.  The community is welcome to bikeshed on the most clear and concise name for this.
