# Partial Initializers

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-partial-initializers.md)
* Author(s): [Matthew Johnson](https://github.com/anandabits)
* Status: **Awaiting review**
* Review manager: TBD

## Introduction

This proposal introduces partial initializers.  They perform part, but not all, of phase 1 initialization for a type.  Partial initializers can only be called by designated initializers of the same type or other partial initializers of the same type.

Swift-evolution thread: [Proposal Draft: Partial Initializers](https://lists.swift.org/pipermail/swift-evolution)

## Motivation

Partial initializers will make it much easier to factor out common initialization logic than it is today.

### Memberwise initialization

Partial initializers are a general feature that can work together with [Parameter Forwarding](https://github.com/anandabits/swift-evolution/edit/parameter-forwarding/proposals/NNNN-parameter-forwarding.md) and [Property Lists](LINK) to enable extremely flexible memberwise initialization.  

The combination of partial initializers and parameter forwarding is sufficiently powerfule to replace the explicit memberwise initializers of the [Flexible Memberwise Initialization](https://github.com/apple/swift-evolution/blob/master/proposals/0018-flexible-memberwise-initialization.md) proposal by simply adding a three implicit partial initializers.

### Extensions with stored properties

Partial initialization is an enabling feature for stored properties in class extensions.  Extension with stored properties would be required to have a designated initializer.  That extension initializer would effectively be treated as a partial initializer by designated initializers of the class.  

John McCall [briefly described](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151221/004479.html) how this might work in the mailing list thread discussing extensions with stored properties. 

## Proposed solution

The proposed solution is to introduce a `partial` declaration modifier for initializers.  

1. When this declaration modifier is present the entire body of the initializer must comply with phase 1 initialization.  It is possible for a partial initializer to initialize all stored properties but it is not possible for a partial initializer to call a `super` initializer.

2. Partial initializers can only be called during phase 1 of initialization by another partial initalizer of the same type, by a designated initializer of the same type, or by another struct initializer of the same type.

3. The compiler keeps track of the properties initialized by a call to a partial initializer and uses that knowledge when enforcing initialization rules in phase 1 in the calling initializer.

4. Partial initializers receive an identifier, which avoids the need to rely on overloading to differentiate between partial initializers.

5. Struct partial initializers are allowed to include an access control modifier specifying their visibility.  They can be called in any initializer of the same type where they are visible.

6. Class partial initializers are always private.  This is because they can only be declared in the main body of a class and can only be called by a designated initializer of the same type.

7. There is no restriction on the order in which the partial initializers are called aside from the rule that the can only be called during phase 1 of initialization and the rule that a `let` property must be initialized once and only once during phase 1.

### Basic example

```swift
struct S {
  let a, b, c, d: Int
  partial init bAndC(i: Int = 10) {
    b = 10
    c = 20
  }
  init(i: Int) {
    a = i * 2
    d = i * 4
    bAndC.init()
  }
}
```

### Calling multiple partial initializers 

```swift
struct S {
  let a, b, c, d: Int
  partial init bAndC() {
    b = 10
    c = 20
  }
  partial init configureD(i: Int) {
    d = i * 4
  }
  init(i: Int) {
    configureD.init(i)
    a = i * 2
    bAndC.init()
  }
}
```

### One partial init calling another

```swift
struct S {
  let a, b, c, d: Int
  partial init bAndC() {
    b = 10
    c = 20
  }
  partial init bcAndD(i: Int) {
    d = i * 4
    bAndC.init()
  }
  init(i: Int) {
    a = i * 2
    bcAndD.init(i)
  }
}
```

### Syntactic sugar for forwarding

It will be a common use case to factor out some common initialization logic using a partial initializer.  Often the parameters for the partial initializer will simply be forwarded.  

Syntactic sugar is provided to streamline this use case.  It matches the placeholder syntax of the [parameter forwarding proposal](https://github.com/anandabits/swift-evolution/blob/parameter-forwarding/proposals/NNNN-parameter-forwarding.md), with the placeholder identifier matching the name of a partial initializer.  The implementation of forwarding matches the implementation of the parameter forwarding proposal.

#### Basic forwarding sugar example

```swift
struct S {
  let a, b, c, d: Int
  partial init bAndC(b: Int = 10, cLabel c: Int = 20) {
    b = b
    c = c
  }
  
  // user writes
  init(i: Int, ...bAndC) {
    a = i * 2
    d = i * 4
  }
  
  // compiler synthesizes
  init(i: Int, b: Int = 10, cLabel: Int = 20) {
    bAndC.init(b: b, cLabel: cLabel)
    a = i * 2
    d = i * 4
  }
  
  // equivalent to writing the following under the parameter forwarding proposal:
  // NOTE: the placeholder identifier is changed to `bAndCParams` here to avoid
  // conflict with the name of the partial initializer itself
  init(i: Int, ...bAndCParams) {
    bAndC.init(...bAndCParams)
    a = i * 2
    d = i * 4
  }
}
```

#### Forwarding to more than one partial initializer

```swift
struct S {
  let a, b, c, d: Int
  partial init bAndC(b: Int = 10, cLabel c: Int = 20) {
    b = b
    c = c
  }
  partial init aAndD(i: Int) {
    a = i * 10
    d = i * 100
  }
  
  // user writes
  init(...aAndD, ...bAndC) {}
  
  // compiler synthesizes
  init(i: Int, b: Int = 10, cLabel: Int = 20) {
    aAndD.init(i: Int)
    bAndC.init(b: b, cLabel: cLabel)
  }
  
  // equivalent to writing the following under the parameter forwarding proposal:
  // NOTE: the placeholder identifier is changed to `bAndCParams` here to avoid
  // conflict with the name of the partial initializer itself
  init(...aAndDParams, ...bAndCParams) {
    aAndD.init(...aAndDParams)
    bAndC.init(...bAndCParams)
  }
}
```

#### One partial initizer forwarding to another

```swift
struct S {
  let a, b, c, d: Int
  partial init bAndC(b: Int = 10, cLabel c: Int = 20) {
    b = b
    c = c
  }
  
  // user writes
  partial init abAndC(...bAndC) {
    a = 42
  }
  
  // compiler synthesizes
  partial init abAndC(b: Int = 10, cLabel c: Int = 20) {
    bAndC.init(b: b, c: c)
    a = 42
  }
  
  // equivalent to writing the following under the parameter forwarding proposal:
  // NOTE: the placeholder identifier is changed to `bAndCParams` here to avoid
  // conflict with the name of the partial initializer itself
  partial init abAndC(i: Int, ...bAndCParams) {
    bAndC.init(...bAndCParams)
    a = 42
  }
}
```

#### Forwarding to super

If super contains a single designated initializer subclasses can use the same syntax to forward parameters to the super initializer.  The call to super is added at the end of the initializer body.  This means that if phase 2 initialization logic is necessary it will not be possible to use the syntactic sugar.

```swift
class Base {
  init(i: Int, s: String, f: Float) {}
}

class Derived: Base {
  let d: Double
  
  // user writes
  init(...super) {
    d = 42
  }
  
  // compiler synthesizes
  init(i: Int, s: String, f: Float) {
    d = 42
    super.init(i: i, s: s, f: f)
  }
  
  // equivalent to writing the following under the parameter forwarding proposal:
  init(...superParams) {
    d = 42
    super.init(...superParams)
  }
}
```

If super contains more than one initializer

```swift
struct S {
  let i: Int, s: String, f: Float
}

class Base {
  init(s: S) {}
  init(i: Int, s: String, f: Float) {}
}

class Derived: Base {
  let d: Double = 42
  // error: ambiguous forward to super
  init(...super) {} 
}
```

### Implicit partial initializers

Three implicit paritial initializers exist.  They match the behavior of `public`, `internal`, and `private` memberwise intializers using the automatic property eligibility model described in the [Flexible Memberwise Initialization](https://github.com/apple/swift-evolution/blob/master/proposals/0018-flexible-memberwise-initialization.md) proposal, thus making that proposal obsolete if this proposal is accepted.  The `private` and `internal` implicit partial initializers also match the behavior of the implicit memberwise initializer if one exists for the type.

```swift
  // flexibile memberwise initialization proposal:
  public memberwise init(...) {
    // init all private an internal props
  }
  // corresponding syntax using implicit partial init and forwarding:
  public init(...publicMemberwise) {
    // init all private an internal props
  }
  
  // flexibile memberwise initialization proposal:
  memberwise init(...) {
    // init all private props
  }
  // corresponding syntax using implicit partial init and forwarding:
  init(...internalMemberwise) {
    // init all private props
  }
  
  // flexibile memberwise initialization proposal:
  private memberwise init(...) {}
  // corresponding syntax using implicit partial init and forwarding:
  private init(...privateMemberwise) {}
```

## Detailed design

TODO but should fall out pretty clearly from the proposed solution

## Impact on existing code

This is a strictly additive change.  It has no impact on existing code.

## Alternatives considered

I believe the basic structure of partial initialization falls naturally out of the current initialization rules.

The syntax for declaring and invoking partial initializers is game for bikeshedding.

### Members computed tuple property

Joe Groff posted the idea of using a [`members` computed tuple property](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160104/005619.html) during the review of the [Flexible Memberwise Initialization](https://github.com/apple/swift-evolution/blob/master/proposals/0018-flexible-memberwise-initialization.md) proposal.  The ensuing discussion inspired me to think more deeply about how more general features could support the memberwise initialization use case.  That line of thinking eventually led me to create this proposal as well as the [Property Lists](LINK) proposal.

There are a few problems with the `members` computed tuple approach:

1. It uses a computed property setter to initialize `let` properties.  This is not something you can do in manually written code and just feels wrong.  Initialization, especially of `let` properties, but also the first `set` of a `var` property, should happen in an initilizer context.  Partial initializers allow for that in a much more elegant fashion than a weird special case property with a setter that is kind of an initializer.
2. The question of how to expose default property values in initializer parameters was never answered.
3. The question of how to provide memberwise initialization for a subset of properties was never answered.
