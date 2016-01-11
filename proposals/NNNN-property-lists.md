# Property Lists

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-property-lists.md)
* Author(s): [Matthew Johnson](https://github.com/anandabits)
* Status: **Awaiting review**
* Review manager: TBD

## Introduction

This proposal introduces the `propertylist` declaration.  Property lists provide concise syntactic sugar for declaring memberwise partial initializers and memberwise computed tuple properties.

NOTE: I do not love the name "property lists" for the feature or the keyword but haven't thought of anything better.  Suggestions are welcome.

Swift-evolution thread: [Proposal Draft: Property Lists](https://lists.swift.org/pipermail/swift-evolution)

## Motivation

I believe the review of the [Flexible Memberwise Initialization](https://github.com/apple/swift-evolution/blob/master/proposals/0018-flexible-memberwise-initialization.md) proposal demonstrated a strong demand for concise yet flexible memberwise initialization.  The discussion highlighted several areas where that proposal fell short:

1. Clarity regarding which parameters receive memberwise initialization.
2. Control over which parameters receive memberwise initialization.
3. Control over specific memberwise parameter ordering.
4. Control over parameter labels.
5. Control over default parameter values, especially for `let` properties.
6. It is a very narrow, special case feature.

This proposal builds on the [Partial Initializer](https://github.com/anandabits/swift-evolution/blob/partial-initializers/proposals/NNNN-partial-initializers.md) proposal to solve these problems using a more general underlying mechanism.  It enables truly flexible memberwise initialization. 

Property lists also support other memberwise features, beginning with memberwise computed tuple properties.

## Proposed solution

There are two ways to specify a property list.  The basic mechanism is a property list declaration that looks as follows: 

```swift
  propertylist configProperties: aPropName, customLabel anotherPropName = 42
```

All properties mentioned in the property list declaration must be visible at the site of the declaration.

A `propertylist` attribute also exists for property declarations to support cases where a list of property declarations should also be treated as a property list:

```swift
  @propertylist(aPropertyListIdentifier) let prop1, prop2: Int
```

A property list can contain any kind of property, including those with behaviors.  However, when certain kinds of properties are included it will not be possible to synthesize a partial memberwise initializer and / or a setter for the computed tuple property.

### Synthesized elements

A typealias will always be synthesized:

1. It will have a name matching the identifier of the property list, with the first character transformed to upper case.
2. It will be a tuple type containing labels and types matching those specified in the property list.

A computed tuple property will always be synthesized:

1. It will always include a getter. 
2. Visibility of the getter will match the visibility of the least visible getter.
3. It will contain a setter as long as all of the properties are settable.
4. Visibility of the setter will match the visibility of the least visible setter.

A paritial initializer will only be generated if:

1. All properties are stored properties.
2. None are `let` properties with an initial value.
3. None have a behavior which is incompatible with phase 1 initialization.
4. Visibility of the partial initializer will match the least visible setter for structs.  Partial initializers for classes are always private (as specified by the partial initializer proposal).
5. The property list is declared in the main body of the type, not an extension, **unless** the type is a struct.
6. If stored properties are allowed in extensions and / or protocols in the future, all properties included in the list must be declared within the same body as the property list for a partial initializer to be synthesized (either the main body of the type or the body of the same extension or same protocol).

The structure of the synthesized elements is as follows:

1. Ordering of partial initializer parameters and tuple members will match the order of the property list declaration.
2. The external parameter labels and tuple labels will match the label specified in the property list if it exists and the property name otherwise.
3. The default value for partial initializer parameters will be the default value specified in the property list if it exists and the initial value of the property otherwise (if that exists).  If neither exist the parameter will not have a default value.

Visibility of a synthesized members is capped at `internal` unless `public` is explicitly specified.  If `public` (or `internal`) is explicitly specified, all properties referenced must have getter and setter visibility of **at least** the specified access level or a compiler error will result.

### Examples

#### Basic example

```swift
public struct S {
  let i: Int = 42
  public var s: String = "hello"
  public var d: Double
  
  // user declares:
  public propertylist custom: dLabel d = 42, s
  
  // compiler synthesizes:
  public typealias Custom = (dLabel: Double, s: String)
  public var custom: Custom {
    get { return (dLabel: d, s: s) }
    set { (d, s) = newValue }
  }
  public partial init custom(dlabel d: Double = 42, s: String = "hello") {
    self.d = d
    self.s = s
  }
}
```

#### Including a let with a initial value

```swift
struct S {
  let i: Int = 42
  let s: String = "hello"
  var d: Double
  
  // user declares:
  propertylist custom: dLabel d, s
  
  // compiler synthesizes:
  typealias Custom = (dLabel: Double, s: String)
  var custom: Custom {
    get { return (dLabel: d, s: s) }
  }
  // NOTE: no setter because a `let` was included
  // and no partial initializer because the `let` 
  // has an initial value.
}
```

#### Including a lazy property

```swift
struct S {
  let i: Int
  var s: String
  lazy var d: Double
  
  // user declares:
  propertylist custom: dLabel d, s
  
  // compiler synthesizes:
  typealias Custom = (dLabel: Double, s: String)
  var custom: Custom {
    get { return (dLabel: d, s: s) }
    set { (d, s) = newValue }
  }
  // NOTE: no partial initializer because a `lazy var` was included.
}
```

#### Including a `var` with a private setter


```swift
struct S {
  let i: Int
  var s: String
  private(set) var d: Double = 42
  
  // user declares:
  propertylist custom: dLabel d, s = "hello"
  
  // compiler synthesizes:
  typealias Custom = (dLabel: Double, s: String)
  private(set) var custom: Custom {
    get { return (dLabel: d, s: s) }
    set { (d, s) = newValue }
  }
  // NOTE: The initial value for `d` is used as a default 
  // parameter value because a different default parameter value 
  // was not specified.
  private partial init custom(dlabel d: Double = 42, s: String = "hello") {
    self.d = d
    self.s = s
  }
}
```

#### Including a computed property

```swift
struct S {
  let i: Int
  var s: String
  var d: Double {
    get { return getValueFromSomewhere() }
    set { storeValueSomewhere(newValue) }
  }
  
  // user declares:
  propertylist custom: dLabel d, s
  
  // compiler synthesizes:
  typealias Custom = (dLabel: Double, s: String)
  var custom: Custom {
    get { return (dLabel: d, s: s) }
    set { (d, s) = newValue }
  }
  // NOTE: no partial initializer because a computed property was included.
}
```

#### Using the `@propertylist` attribute

```swift
struct S {
  @propertylist(custom) var s: String, d: Double = 42
  private let i: Int
  
  // compiler synthesizes:
  typealias Custom = (s: String, d: Double)
  var custom: Custom {
    get { return (s: s, d: d) }
    set { (s, d) = newValue }
  }
  partial init custom(s: String, d: Double = 42) {
    self.s = s
    self.d = d
  }
}
```

#### Using a property list declaration for memberwise intialization

```swift
struct S {
  var x, y, z: Int
  let s: String
  let d: Double
  
  propertylist randomGroup aString s = "hello", anInt x = 42
  
  // user declares
  init(...randomGroup) {
    y = 42
    z = 42
    d = 42
  }
  
  // compiler synthesizes
  partial init randomGroup(aString s: String = "hello", anInt x: Int = 42) {
    self.s = s
    self.x = x
  }
  init(aString s: String = "hello", anInt x: Int = 42) {
    randomGroup.init(aString: s, anInt: x)
    y = 42
    z = 42
    d = 42
  }
  
}
```

#### Using the property list attribute for memberwise intialization

```swift
struct S {
  @propertylist(declaredTogether) var x, y, z: Int
  let s: String
  let d: Double
  
  // user declares:
  init(...declaredTogether) {
    s = "hello"
    d = 42
  }
  
  // compiler synthesizes:
  partial init declaredTogether(x: Int, y: Int, z: Int) {
    self.x = x
    self.y = y
    self.z = z
  }
  init(x: Int, y: Int, z: Int) {
    declaredTogether.init(x: Int, y: Int, z: Int)
    s = "hello"
    d = 42
  }
  
}

### Implicit property lists

It may be desirable to have several implicit property lists available, such as one that includes all properties of the type.  Properties would appear in any implicit property lists in declaration order.

The specific details of what implicit property lists should be available and what they should be called is a good topic for bikeshedding.

## Detailed design

TODO but should fall out pretty clearly from the proposed solution

## Impact on existing code

This is a strictly additive change.  It has no impact on existing code.

## Alternatives considered

We could live without this syntactic sugar. There are several reasons the language is better with it:

1. It is much more convenient than manually writing memberwise partial intializers.  It does not include any unnecessary information, thus making the details more clear.  
2. It gives us the memberwise computed tuple properties for free.  This would would not be possible when writing memberwise partial initializers manually.
3. It scales to any new memberwise features that might make sense for a type.

As always, the `propertylist` keyword is game for bikeshedding.  The use of `:` to separate the identifier from the list of properties could also be game for bikeshedding.  Also, as mentioned previously, the implicit property lists are game for bikeshedding.
