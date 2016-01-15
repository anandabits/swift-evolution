# Partial Initializers

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-partial-initializers.md)
* Author(s): [Matthew Johnson](https://github.com/anandabits)
* Status: **Awaiting review**
* Review manager: TBD

## Introduction

This proposal introduces partial initializers and initialization methods.  They perform part, but not all, of phase one initialization for a type.  They can be called by designated initializers or other partial initializers.  Initialization methods may also be called like normal methods post-initialization, but may only write to `var` properties.

Swift-evolution thread: [Proposal Draft: partial initializers](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160104/006179.html)

## Motivation

Swift currently requires all initialization assigments to happen in the main body of a designated initializer.  Swift does this for a great reason: to enforce rules enabling definitive and safe initiailization.  

Unfortunately this means that we are **required** to separate any computation we wish to factor out from the initialization assignments.  This may be a good idea in some cases, but in others it is inconvenient and leads to less readable code.  

Lifting this limitation while still guaranteeing definitive and safe initialization is very desirable.  It will allow us to better factor out common behavior and make our code more readable.  It will also lay a foundation that could be built upon by future language features.

### Factoring initialization logic

We can currently factor out computation involve in initialization using a static method like this:

```
struct S {
    var a, b: Int
    static func computeValues(i: Int) -> (Int, Int) {
        return (i * 2, i * 4)
    }
    mutating func resetValues(i: Int) {
        (a, b) = S.computeValues(i)
    }
    init(i: Int) {
        (a, b) = S.computeValues(i)
    }
}
```

Separating the computation from the assignment obfuscates the simple nature of what is happening.  We ae also required to duplicate the call and assignment in both the initializer in the method.  An initialization method makes the code much more clear while also eliminating the duplication.

```
struct S {
    var a, b: Int
    init func resetValues(i: Int) {
      a = i * 2
      b = i * 4
    }
    init(i: Int) {
        resetValues.init(i)
    }
}
```

### Extensions with stored properties

This proposal **does not** introduce extensions with stored properties, however partial initialization is closely related to that topic.  Class extensions with stored properties would be required to have an extension initializer.  That extension initializer would effectively be treated as a partial initializer by designated initializers of the class.  

One interesting consequence to note is that if we have class extensions with stored properties but do not have partial initializers there will be a tension between the ability to organize properties as desired and the the ability to use a partial initializer factor out their initialization.

John McCall [briefly described](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151221/004479.html) how initialization of class extensions with stored properties might work in the mailing list thread discussing extensions with stored properties.  

He also [requested](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160111/006261.html) that I include in this proposal details regarding how partial initializers are related to extensions with stored properties.  I discuss this in the "Interactions with other features" section at the end of this proposal.

### Memberwise initialization

The core team review of the [Flexible Memberwise Initialization](https://github.com/apple/swift-evolution/blob/master/proposals/0018-flexible-memberwise-initialization.md) proposal concluded with the recommendation that an "opt-in" model is the most promising direction.  I agree with this conclusion.  This proposal **does not** introduce anything specific to memberwise initialization but it could play an enabling role for the design of an "opt-in" model when it is time to take another look at memberwise initialization.

One topic that received quite a bit of discussion on the list during the review is whether it might be possibe for more general features to support memberwise intialization rather than building a specific feature and syntax supporting it directly into the language.  Definitive initialization imposes strict rules and requirements on any potential solution (esepcially for `let` properties).  This makes it difficult or impossible to properly support memberwise intialization with completely general features.  

However, I do believe it can be well supported by more general initialization features.  Partial initializers are a general initialization feature that can play an important role in doing so.  There are several possible designs for such a solution.

John McCall [requested](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160111/006261.html) that I include in this proposal details regarding how partial initializers are related to memberwise initialization.  I discuss the design options for this further in the "Interactions with other features" section at the end of this proposal.

## Proposed solution

The proposed solution is to introduce partial initializers and (partial) initializer methods.  Because they only initialize part of an instance they are given an identifier that can be used to describe the role they play in initialization.

Partial initalizers have the form `init aPartialInitializerName() {}`

Initializer methods have the form `init func anInitializationMethodName() {}`

During initialization they are used as follows: 

```swift
  aPartialInitializerName.init()
  anInitializationMethodName.init()
```

Both partial initializers and initialization methods must adhere to the rules that would be applied if the code they contain was placed at the very beginning of phase one of a designated initializer.  See the detailed design section for the complete rules.

The only difference between them is that initializer methods may be called like any ordinary method post-initialization (for example, to reset the state of an instance), but may only initialize `var` properties.  Because initialization inherently involves mutation, initializer methods are implicitly considered `mutating` methods when declared as part of a struct.

Throughout the rest of this proposal the term **partial initializers** refers to both partial initializers **and** initialization methods unless the distinction is important to the context.

### Basic partial initializer example

```swift
struct S {
  let a, b, c, d: Int
  init bAndC(i: Int = 10) {
    b = i * 10
    c = i * 20
  }
  init(i: Int) {
    a = i * 2
    d = i * 4
    bAndC.init(i)
  }
}
```

### Basic initializer method example

```swift
struct S {
  var a, b, c, d: Int
  let i: Int
  init func bAndC(i: Int = 10) {
    b = 10
    c = 20
  }
  init func aAndD(i: Int) {
    a = i * 2
    d = i * 4
  }
  init(i: Int) {
    self.i = i
    bAndC.init(i)
    aAndD.init(i)
  }
  mutating func reset() {
    aAndD(i)
    bAndC(i)
  }
}
```

### Calling a partial initializer and an initializer method

```swift
struct S {
  var a, b, c, d: Int
  init func bAndC() {
    b = 10
    c = 20
  }
  init configureD(i: Int) {
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
  init bAndC() {
    b = 10
    c = 20
  }
  init bcAndD(i: Int) {
    d = i * 4
    bAndC.init()
  }
  init(i: Int) {
    a = i * 2
    bcAndD.init(i)
  }
}
```

## Detailed design

### Partial initializers

1. The entire body of a partial initializer or initialization method must comply with phase one initialization.  It is possible for a partial initializer or initialization method to initialize all stored properties but it is not possible for a partial initializer to call a `super` initializer.

2. Partial initializers receive an identifier, which avoids the need to rely on overloading to differentiate between partial initializers and allows for a name describing the role of the partial initializer.  Overloading identifiers is allowed as it is throughout the language.

3. Partial initializers and initialization methods may include an access control modifier specifying their visibility.

4. Neither partial initializers nor initialization methods are allowed to throw.  This may be relaxed in time, but John McCall mentioned that it will make the initial implementation more straightforward.

5. Struct partial initializers and initialization methods may be declared in the main body of the struct or in an extension, but may only initialize stored properties that are visible at the declaration site.

6. Struct partial initializers and initialization methods may be called in any initializer (partial or not) of the same type (as long as they are visible).

7. Class partial initializers and initialization methods may be only be declared in the main body of the class.

8. Class partial initializers and initialization methods may only be called during initialization by another partial initializer of the same type or a designated initializer of the same type while it is in phase one of initialization.

9. The compiler keeps track of the properties initialized by a partial initializer or initialization method and uses that knowledge when enforcing initialization rules in phase one in the calling initializer.

    1. A partial initializer can initialize any subset of stored properties (including all of them).
    2. An initialization method can initialize any subset of `var` stored properties (including all of them).
    3. A partial initializer or initialization method can only read a property it initializes itself.
    4. A partial initializer or initialization method can only write to a property once, including `var` properties.
    5. An initializer is not allowed to call a partial initializer or initialization method that initializes a property that has already been initialized (other than the initial value of a `var`, which may be overwritten by a partial initializer or initialization method).  This means that redundant initialization of `var` properties is not allowed.
    6. After calling a partial initializer or initialization method, the properties it initialized are considered initialized in the calling initializer as well.  They can be read from, etc.

10. Initialization methods may be called anywhere they are visible with normal method syntax using their identifer after phase one initialization completes.  This allows them to be used to reset the state of an instance if necessary.  They will be considered `mutating` methods when called this way on a `struct` instance.

## Impact on existing code

This is a strictly additive change.  It has no impact on existing code.

## Alternatives considered

The rules adopted by this proposal are intended to represent the minimum functionality necessary to enable partial initializers and initialization methods.  More expansive solutions were not seriously considered as they may introduce unnecessary points of controversy or complexity.  Many of the rules may be relaxed in the future after we have some experience with the basic feature.

The syntax for declaring and invoking partial initializers is game for bikeshedding.  Some syntax alternatives considered follow.

### Require a `partial` declaration modifier

The first draft of this proposal used the syntax `partial init anIdentifier` rather than just `init anIdentifier`.  The declaration modifier was dropped because:

1. No initializers are currently allowed to have an identifier.
2. Identifiers are really only useful for initializers that are partial, where the name describes **what part** of the type it initializes.
3. If the only initializers that contain identifiers are partial intiailizers, the identifier itself distinguishes an initializer as a partial initializer.

### Invoke partial initializers like normal methods

Rather than invoking partial initializers with the `identifer.init()` syntax it would be possible to invoke them with the usual method syntax `indentifier()` or `self.identifier()` during initialization.  I believe the syntactic distinction during initialization is useful because it emphasizes the fact that the call plays a role in the initialization of the instance.

### Invoke initializer methods with a consistent syntax

Initializer methods are invoked with the `identifier.init()` syntax during initialization and the `identifier()` or `self.identifier()` syntax post-initialization.  This inconsistency may bother some people.  Here are the options we have:

1. Use the normal method syntax in both cases.
2. Use the normal method syntax for initialization methods, but the `identifier.init()` syntax for partial initializers during initialization.
3. Use the approach in this proposal.
4. Use the `identifier.init()` syntax for initialization methods **outside** of initialization context as well is during initialization.
5. Do not allow for initialization methods.

The previous alternative considered addresses #1 and the following alternative considered addresses #5.  #2 means an inconsistent usage during initialization, which I believe would be confusing.  #4 seems somewhat strange when invoked inside a method of the type itself and would extremely strange if used outside of an instance method of the type itself.  

The approach adopted by this proposal is the remaining option and seems to have the least downsides.  It would be reasonable to go with #1 instead.  If there is a strong community preference for that I will modify the proposal.

### Do not allow for initializer methods

The first draft of this proposal did not allow for initializer methods.  This oversight was [pointed out](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160111/006328.html) by David Owens II.  The only argument against allowing initializer methods is that it makes the model slightly more complex.  I believe they will be useful enough to pay for that modest incremental complexity.

### Do not allow for partial initializers

If the proposal only included initializer methods, but not partial initializers it would not be possible to factor out initialization logic that must set a `let` property.  If we are going to introduce a feature supporting partial initialization it must work with any stored property.

### Allow partial initializers to read properties they do not initialize

Brent Royal-Gordon has been making a case that we should also allow partial intializers to read from properties that must be initialized prior to calling them during phase one.  This may be useful in some cases but it would also mean that some partial initializers cannot be invoked **anywhere** in phase one.  This is a potential source of confusion, especially when partial initializers are a new concept.

If the core team supports relaxing this requirement I will make the change.  But I do not believe it is a core feature of this proposal and don't want it to be derailed by an attempt to support this.  Many of the rules, including this one, can be relaxed in time as we gain experience with partial initializers.

### Members computed tuple property

Joe Groff posted the idea of using a [`members` computed tuple property](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20160104/005619.html) during the review of the [Flexible Memberwise Initialization](https://github.com/apple/swift-evolution/blob/master/proposals/0018-flexible-memberwise-initialization.md) proposal.  The ensuing discussion inspired me to think more deeply about how more general features could support the memberwise initialization use case.  This line of thinking eventually led to this proposal.

There are a few problems with the `members` computed tuple approach:

1. It uses a computed property setter to initialize `let` properties.  This is not something you can do in manually written code and just feels wrong.  Initialization, especially of `let` properties, but also the first `set` of a `var` property, should happen in an initilizer context.  Partial initializers allow for that in a much more elegant fashion than a weird special case property with a setter that is kind of an initializer.
2. The question of how to expose default property values in initializer parameters was never answered.
3. The question of how to provide memberwise initialization for a subset of properties was never answered.

The core team also rejected this approach in their review of the Flexible Memberwise initialization proposal.

## Interactions with other features

This section is included at the request of John McCall.  He thought it would be useful to clarify how partial initializers are be related to other features that may be added to Swift in the future.  

If you skip this section you will not miss anything important regarding what is actually being proposed.  At most, you may miss some context for the motivation of this feature.  However, partial intializers and intialization methods should be compelling enough to adopt on their own if they are to be accepted.  

If you only interested in what is actually being proposed you should skip the rest of this document.

As it is quite lengthy, it may be a good idea to extract this section to a separate document and link to it.  I would like feedback on the best way to handle this.

### Class extensions with stored properties

Several aspects of this proposal are closely related to introducing stored properties in class extensions.  I am providing a basic sketch of how such a feature might interact with this proposal.  This proposal **does not** introduce class extensions with stored properties and may not reflect the proposal that eventually does introduce that feature.

Class extensions with stored properties would be allowed to include two kinds of partial initializers:

1. Partial initializers which initialize all stored properties declared in the extension.  These would be called complete extension initializers and specified with the `extension init() {}` syntax.  Complete extension initializers can be thought of as a designated initializer for the extension.  As such, they do not receive an identifier and may not be initialization methods.

2. Partial initializers and initialization methods which initialize a subset of the stored properties declared in the extension.  These would be exactly like partial initializers in the main body of the class, but with respect to the complete extension initializers rather than designated initializers.

If the class extension includes initial values for all stored properties it introduces it would receive a default complete extension intializer if no complete extension initializers are explicitly declared for the extension.

Rules for initializing the extension might look like this:

1. Class extensions with stored properties must include an identifier naming the extension.
2. Designated initializers for the class must call exactly one complete extension initializer for each extension containing stored properties, unless the extension includes an initializer that does not require any arguments, in which case the call may be omitted and a call to the no-argument initializer synthesized at the beginning of the designated initializer (similar to how a call to `super` may be omitted, except the synthesized call to the complete extension initializer will be emitted at the beginning of the designated initializer rather than at the end).  

Example:

```swift
extension MyClass MoreStuff {
  let a, b: Int
  init onlyA() {
    a = 44
  }
  extension init(i: Int) {
    onlyA.init()
    b = i * 2
  }
}

class MyClass {
  let x, y: Int
  init() {
    x = 2
    y = 2
    MoreStuff.init(44)
  }
}
```

It is possible that a similar strategy could be used for initializing protocols with stored properties, depending on the details of how that feature was designed.

In general, the interaction of partial initialization with any new features introducing stored properties in new contexts would need to be carefully considered.

### Memberwise initialization

This section lays out some options for designing support for memberwise initialization on top of more general initialization features.  All options require both partial initializers as well as another general initialization feature: initializer forwarding.  

#### Initializer Forwarding

The basic idea behind initializer forwarding is that if we have a partial initialization feature it will be extremely common to pass parameters directly from the designated initializer to a partial initializer.  Essentially, initializer forwarding is a concise way to compose partial initializers into a complete initializer.  

It is the concise composition of initializers that could play a supporting role in memberwise initialization where the designated initializer would forward to a memberwise partial initializer.  Of course, it would also be useful in many cases that do not involve memberwise initialization, or involve composing a memberwise partial initializer with a manual partial initializer.

The syntax I will use in the examples allows a placeholder that looks like `...partialInitializerIdentifier` to be used in the parameter list of an initializer.  The identifier in the placeholder references a partial initializer or initialization method.  The reference must be unambiguous.  If Doug's proposal Naming Functions with Argument Labels is accepted, presumably we would also be able to use the complete name of a partial initializer in the placeholder if necessary for disambiguation.  It would not be possible to forward to partial initializers overloaded by type only.

The placeholder would be expanded to by the compiler to the complete parameter list of the forwardee, including default values, as well as the forwarding call at the beginning of the initializer body.

The following example is contrived, but it demonstrates the basic idea of initializer forwarding:

```swift
struct S {
  let a, b, c, d: Int
  init aAndB(a: Int = 10, bLabel b: Int = 20) {
    a = a
    b = b
  }
  
  init cAndD(c: Int = 10, dLabel d: Int = 20) {
    c = c
    d = d
  }
  
  // user writes
  init(...aAndB, ...cAndD) {}
  
  // compiler synthesizes
  init(a: Int = 10, bLabel b: Int = 20, c: Int = 10, dLabel d: Int = 20) {
    aAndB.init(a: a, bLabel: b)
    cAndD.init(c: c, dLabel: d)
  }
  
  // equivalent without partial initializers
  init(a: Int = 10, bLabel b: Int = 20, c: Int = 10, dLabel d: Int = 20) {
    a = a; b = b
    c = c; d = d
  }
}
```

It is possible to compose partial initializers manually.  However, in simple cases doing this actually results in more code than avoiding partial initializers altogether.  The manual approach also makes it much less clear that we are simply composing two partial initializers.

As Chris has pointed out a number of times, pure sugar features really need to pay their way in terms of making code more concise and readable.  Any solution for memberwise initialization must pass this test.  The ability to compose parital initializers with forwarding enables them to pass the test.

Partial initializers on their own are not sufficient to support memberwise initialization.  When combined with initializer forwarding several design approaches become possible.  The rest of this section describes several possible designs.  They are not mutually exclusive.  There is significant overlap so we would not want to adopt all of them, but it might make sense to support more than one.

#### Implicit partial initializer

The most basic way to build upon partial initializers and initializer forwarding is to introduce an implicit partial initializer that simply initializes all stored properties, using any initial values as default parameter values.  

The following example would explicitly request an initializer equivalent to the current implicit memberwise initializer:

```
struct S {
  var x, y, z: Int
  init(...members) {}
}
```

There is ample opportunity to bikeshed over the name of this implicit partial initializer.  Some options would be `members`, `memberwise` or `allProperties`.

I believe it is a good idea to include this implicit partial initializer if we retain an implicit memberwise initializer.  It allows us to explicitly declare the implicit memberwise intializer.  

This implicit partial initializer would be almost essential if we do not retain the implicit memberwise initializer as it would partly mitigate the boilerplate necessary to declare an initializer explicitly in the "bag of properties" use case.

#### Property declaration modifier

This approach is very similar to the "opt-in" version of my proposal the core team discussed in their feedback.  

Instead of using a `memberwise` declaration modifier on the initializer, it would forward to an implicit memberwise partial initializer that is synthesized based on the property declaration modifier.

The example in the core team feedback would look like this:

```swift
class C : Derived {
  mwi var x: Int
  mwi var y = 4.0
  mwi var z = "foo"
  var extra = 4.0   // Doesn’t participate in memberwise inits.

  init(q: Int, ...memberwise) {
    // your custom code can go here
    super.init(q)
  }
}
```

The biggest drawback to this approach is that it does not address one of the items in "common feedback and points" presented by the core team (which I agree is an important concern):

    Provide a predictable model.  It was concerning to many people 
    that you could accidentally reorder properties and break a 
    memberwise init without knowing/thinking about it.  
    This directly contravenes the resilience model.  
    API diff’ing tools and “synthesized header” views in Xcode 
    ameliorate this, but don’t totally solve it.

The signature of the initializer is still tightly coupled to property declaration order.  The expansion of the placeholder is not stated in its complete form in any single location.  I believe these issues are significant enough that this approach is likely not the right direction.

It also has the limitation that it is only possible to have one memberwise partial initializer.  This limitation would probably be acceptable, but it is lifted by any approach allowing allowing explicit control over memberwise parameter order.

One thing the core team liked about this approach is that the property declaration modifier makes it clear that the initial value of the property may not always be used (and any side-effects of calculating it not executed).  The last section of this document discusses options surrounding this issue.

#### Property declaration modifier with identifier

Several people on the list like the idea of combing property declarations with a memberwise initializer parameter list as is done in Scala and other languages.

I believe a similar, but more Swifty option would be a declaration modifier intended to be used when declaraing properties in a list.  A partial initializer would be synthesized for the properties included in the declaration.  The modifier would accept an identifier to use for the name of the partial initializer.

```swift
class C : Derived {
  mwi(appearanceProps) let foregroundColor: UIColor = UIColor.blackColor(), 
                           backgroundColor: UIColor = UIColor.whiteColor()
  mwi(animationProps) let delay: NSTimeInterval = 0
                       duration: NSTimeInterval = 0.4
  var extra = 4.0   // Doesn’t participate in memberwise init.

  init(q: Int, ...appearanceProps, ...animationProps) {
    // your custom code can go here
    super.init(q)
  }
}
```

This approach partly mitigates the declaration order problem by keeping the property declarations together in a construct that approximates a parameter list.  It isn't quite as concise as Scala but it gets close, is more flexible, and avoids the cons the core team mentioned regarding the Scala approach.

The use cases where this apporach is viable are somewhat limited without a significant (and quite likely undesirable) modification of the property declaration grammar.  It is not possible to declare properties with different declaration modifiers, attributes and variable / constant status in a single declaration.  

On the other hand, in many builder pattern use cases the properties we would like to include in memberwise initialization **will** share the same attributes, modifiers, and variable / constant status.

I have mixed feelings about this approach; I like that it is very concise but at the same time it is definitely **not** sufficient on its own.  I am not sure whether it is worthwhile to support this **in addition** to other approaches but it is definitely not the right approach for an initial solution.

#### "self.x = x" sugar

The "self.x" syntactic sugar combines niceley with partial initializers and initializer forwarding.  

It gives us full control over the parameter list.  It could support (depending on the final design):

1. Inclusion of any properties regardless of declation modifier, etc (obviously)
2. Explicit control of parameter ordering (obviously)
3. Explicit control of external labels
4. Ability to provide a default parameter value for properties with no initial value or a default that is different than the "initial" value of a property
5. Possibly allowing the parameter for optional properties to be specified as non-optional

Another nice attribute of this solution noted by the core team is that it could apply to any parameter of any initializer.  It is not specific to memberwise intialization.  

David Owens II also pointed out that this syntax could also be used with `var` properties in any method, although in that case initial values would probably not be used as default parameter values

Here is an example:

```
class C : Derived {
  var x: Int
  var y = 4.0
  var z = “foo"

  init xAndY(self.x, self.y) {}
  
  init(q: Int, ...XAndY, self.z) { 
    // your custom code can go here
    super.init(q)
  }
}
```

As the core team also noted, the "self." parameter syntax is a bit strange.  We could probably find something acceptable and people would get used to it.

#### Additional "self.x =x" sugar

The "self.x" syntactic sugar does allow for more concise declaration of memberwise partial initializers.  However, in that use case it results in a repetition of "self." for each property.  

```swift
  init animationProps(self.prop1, self.prop2, self.prop3, self.prop4, self.prop5) {}
```

If this form of sugar plays a role in supporting memberwise initialization it would be nice if we could state that **all** parameters are "property setting parameters".  I am not sure what the best syntax for this would be.  One option is a `self` declaration modifier on the initializer (or method if we support this syntax for `var` properties on methods):

```swift
  self init animationProps(prop1, prop2, prop3, prop4, prop5) {}
```

This syntax communicates the idea and isn't terrible.  As with "self.x" syntax itself, we might be able to find something better.

With syntax like this we are able to declare a partial memberwise initializer very concisely while retaining full control over property inclusion, ordering, and parameters as noted in the previous section.

A complete example using this syntax might look like this:

```swift
class C {
  let prop1: NSTimeInterval = 0
  let prop2: NSTimeInterval = 0
  let prop3: NSTimeInterval = 0
  let prop4: NSTimeInterval = 0
  let prop5: NSTimeInterval = 0
  
  let internalState: Something
    
  self init animationProps(externalLabel prop2, prop3, prop1 = 42, prop5 = 42, prop4 = 4) {}
  
  init(...animationProps) {
    internalState = someValue
  }
}
```

This feels like a very promising direction.  It uses features that have use cases beyond memberwise initialization, but composes them in such a way that it is nearly as concise as possible if explicit control of the parameter list (especially the included properties and their order in the parameter list) is a requirement.

#### General memberwise features

One final possibility is to adopt a general "memberwise" syntax that would support memberwise features beyond just initialization.  The only additional such feature I currently know of would be memberwise computed tuple properties, however there may be others identified in the future.  

The syntax would be very similar to the previous example, possibly something like this:

```swift
  memberwise animationProps = externalLabel prop2, prop3, prop1 = 42, prop5 = 42, prop4 = 4
```

The syntax would allow for the same control over parameters as the previous approach.

A partial initializer as well as a computed tuple property would be synthesized, containing the specified properties.

I shared an early draft of a proposal describing a feature like this.  I called it [Property Lists](https://github.com/anandabits/swift-evolution/blob/property-lists/proposals/NNNN-property-lists.md).  "memberwise" is probably a much better name if we were to go in this direction (it seems obvious but didn't occur to me at the time for some reason).  If you're interested in more details about what this approach might look like, please see that proposal draft, but assume the feature and keyword would be `memberwise` rather than `propertylist`.

While this approach is interesting, I think it only makes sense if there are compelling use cases for the computed tuple properties and / or other memberwise features.  When viewed strictly through the lens of initialization, it only saves a few keystrokes over the previous "Additional self.x =x sugar" while not being useful in non-memberwise initializers.  It is also likely that something like this a good candidate for a macro which could expand to the "self.x" approach.

#### Default vs initial values

The core team noted a potential concern regarding zapping side effects and breaking `let` axioms in regards to treating initial values as default values that may not be used.  

My opinion is that this issue should be treated as orthogonal to memberwise initialization.  There may be cases where we wish to initialize a property that has a "default" value to something other than that default in a non-memberwise initializer.  Restricting this capability to memberwise initializers seems less than ideal IMO.

We have several options here:

1. Current behavior.  Let properties with initial values cannot be otherwise initialized.  Var properties with initial values will have their side-effect executed twice if the default is used as a default parameter value.
2. Suppress the initial value assignment for any properties that are otherwise initialized by the initializer.  Unfortunately this requires the compiler to scan the body of the initializer before it knows what to do and it is not clear from the signature whether the initial value is used or not.
3. Suppress the initial value assignment when forwarding to a partial initializer that initializes the property.  This has the advantage that the compiler can determine what to do from the signature alone because it knows which properties a partial initializer will initialize.  This is also more clear to developers than #2.
4. Suppress the initial value assignment when the property is initialized by a "self.x" parameter directly, or by forwarding to a partial initializer that does so.  This has the advantage that the compiler can determine what to do from the expanded signature alone without considering the body of partial initializers.  This approach also provides the most clarity thus far as the "self.x" parameters clearly indicate initialization of the property using the value provided to the parameter.
5. Use the same logic as in #2 or #3 but require a declaration modifier on properties that have their initial value assignment suppressed.  The declaration modifier might be something like `def`, and indicates that the "initial value" is merely a default that may or may not be used.
6. Do the same as #4, but also require a declaration modifier on the initializer to indicate that it is allowed to "override" the default values of properties.

The "opt-in" example the core team included in their review of the Flexible Memberwise initialization proposal uses #6 with `mwi` and `memberwise` as the declaration modifiers.  The same approach to this problem is possible without tying the behavior specifically to memberwise initialization. 

The downside of using approach #6 when it is not simultaneously determining the memberwise parameter set is that it becomes somewhat more verbose to combine with a different approach to memberwise initialization.  

Using approach #6 in combination with the "Additional self.x =x sugar" approach to memberwise initialization might look like this:

```swift
class C : Derived {
  def var x: Int
  def var y = 4.0
  def var z = "foo"
  var extra = 4.0   // initial value assignment is always synthesized

  self init props(y, z = "hello", x = 0) {}

  def init(q: Int, ...props) {
    // your custom code can go here
    super.init(q)
  }
  def init(q: SomeOtherType, ...props) {
    // your custom code can go here
    super.init(q)
  }
}
```

The good aspect of this approach is that everything is orthogonal and everything is explicit.  The downside is the repetition of `def`, but it isn't that bad especially if we use a short declaration modifier.

I believe #4 strikes a very good balance of clarity while avoiding the need for repetitive syntax.  There are only a couple of cons:

1. It is not possible to tell whether the initial value of a property may be elided by looking at the property declaration alone.  One must also look at the initializer signatures if it is necessary to know this for sure.  
2. It is not possible to prevent initial value synthesis by initializing the property in the body of an intializer.

The first con can be mitigated by generated headers, documentation, etc which could indicate whether this may happen in some cases.  It would also be possible to allow a declaration modifier which ensures the initial value assignment is **always** synthesized, perhaps `init` if it is considered desirable to guarantee this for certain properties.

I believe the second con is acceptable.  It is always possible to acheive the necessary behavior by avoiding an initial value and taking full control of initialization.

I believe the advantages of using "self.x" parameters to communicate the suppression of initial value assignment point strongly in the direction of using that approach to memberwise initialization in conjunction with partial initializers and initializer forwarding.

Using that approach, the final example looks like this (modulo bikeshedding over syntax details of course):

```swift
class C : Derived {
  var x: Int
  var y = 4.0
  var z = "foo"
  var extra = 4.0

  self init props(y, z = "hello", x = 0) {}

  init(q: Int, ...props) {
    // your custom code can go here
    super.init(q)
  }
  init(q: SomeOtherType, ...props) {
    // your custom code can go here
    super.init(q)
  }
}
```
