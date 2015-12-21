# Flexible Memberwise Initialization

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-flexible-memberwise-initializers.md)
* Author(s): [Matthew Johnson](https://github.com/anandabits)
* Status: **Review**
* Review manager: TBD

## Introduction

The Swift compiler is currently able to generate a memberwise initializer for us in some circumstances however there are currently many limitations to this.  This proposal build on the idea of compiler generated memberwise initialization making it available to any initializer that opts in.

Swift-evolution thread: [Proposal Draft: flexible memberwise initialization](https://lists.swift.org/pipermail/swift-evolution)

## Motivation

When designing initializers for a type we are currently faced with the unfortunate fact that the more flexibility we wish to offer users the more boilerplate we are required to write and maintain.  We usually end up with more boilerplate and less flexibility than desired.  There have been various strategies employed to mitigate this problem.  

Sometimes properties that should be immutable are made mutable and a potentially unsafe ad-hoc two-phase initialization pattern is employed where an instance is initialized and then configured immediately afterwards.  When properties that need to be mutable have a sensible default value they are simply default-initialized and the same post-initialization configuration strategy is employed when the default value is not correct for the intended use.  This results in an instance which passes through several incorrect states before it is correctly initialized for its intended use.

Flexible and concise initialization for both type authors and consumers will encourages using immutability where possible and removes the need for boilerplate from the concerns one must consider when designing the intializers for a type.

Quoting [Chris Lattner](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151130/000518.html):

	The default memberwise initializer behavior of Swift has at least these deficiencies (IMO):
	1) Defining a custom init in a struct disables the memberwise initializer, and there is no easy way to get it back.
	2) Access control + the memberwise init often requires you to implement it yourself.
	3) We don’t get memberwise inits for classes.
	4) var properties with default initializers should have their parameter to the synthesized initializer defaulted.
	5) lazy properties with memberwise initializers have problems (the memberwise init eagerly touches it).

Add to the list “all or nothing”.  The compiler generates the entire initializer and does not help to eliminate boilerplate for any other initializers where it may be desirable to use memberwise intiialzation for a subset of members and initialize others manually.

It is common to have a type with a number of public members that are intended to be configured by clients, but also with some private state comprising implementation details of the type.  This is especially prevalent in UI code which may expose many properties for configuring visual appearance, etc.  Flexibile memberwise initialization can provide great benefit in these use cases, but it becomes immediately useless if it is "all or nothing".  

We need a flexible solution that can synthesize memberwise initialization for some members while allowing the type auther full control over initialization of implementation details.

## Proposed solution

I propose adding a `memberwise` declaration for initializers which allows them to *opt-in* to synthesis of memberwise initialization and a `@nomemberwise` property attribute allowing them to *opt-out* of such synthesis. 

This section of the document contains several examples of the solution in action.  Specific details on how synthesis is performed are contained in the detailed design.

### Replacing the current memberwise initializer

```swift
struct S {
	let s: String
	let i: Int

	// user declares:
	memberwise init() {}
	// compiler synthesizes:
	init(s: String, i: Int) {
		self.s = s
		self.i = i
	}
}
```

### Properties with initial values

```swift
struct S {
	let s: String = "hello"
	let i: Int = 42

	// user declares:
	memberwise init() {}
	// compiler synthesizes:
	init(s: String = "hello", i: Int = 42) {
		self.s = s
		self.i = i
	}
}
```

### Partial memberwise initialization

```swift
struct S {
	let s: String
	let i: Int

	// user declares:
	memberwise init() {
		i = getTheValueForI()
	}
	// compiler synthesizes (suppressing memberwise initialization for properties assigned in the initializer body):
	init(s: String) {
		self.s = s
		// body of the user's initializer remains
		i = getTheValueForI()
	}
}
```

### access control

```swift
struct S {
	let s: String
	private let i: Int

	// user declares:
	memberwise init() {
		// compiler error, i memberwise initialization cannot be synthesized 
		// for i because it is less visible than the initializer itself
	}
}
```

### lazy properties and incompatible behaviors

```swift
struct S {
	let s: String
	lazy var i: Int = InitialValueForI()

	// user declares:
	memberwise init() {
	}
	// compiler synthesizes:
	init(s: String) {
		self.s = s
		// compiler does not synthesize initialization for i 
		// because it contains a behavior that is incompatible with 
		// memberwise initialization
	}
}
```

### @nomemberwise properties

```swift
struct S {
	let s: String
	@nomemberwise let i: Int

	// user declares:
	memberwise init(configuration: SomeTypeWithAnIntMember) {
		i = configuration.intMember
	}
	// compiler synthesizes:
	init(configuration: SomeTypeWithAnIntMember, s: String) {
		self.s = s
		i = configuration.intMember
	}
}
```

### uninitialized @nomemberwise properties

```swift
struct S {
	let s: String
	@nomemberwise let i: Int

	// user declares:
	memberwise init() {
		// compiler error, i is not initialized
	}
}
```

### delegating and convenience initializers

```swift

struct S {
	let s: String = "hello"
	let i: Int = 42

	// user declares:
	memberwise init() {}
	// compiler synthesizes:
	init(s: String = "hello", i: Int = 42) {
		self.s = s
		self.i = i
	}
	
	// user declares:
	memberwise init(describable: CustomStringConvertible) {
		self.init(s: describable.description)
	}
	// compiler synthesizes (adding forwarded memberwise parameters):
	init(describable: CustomStringConvertible, i: Int = 42) {
		self.init(s: describable.description, i: i)
	}
}
```

### subclass initializers

```swift
class Base {
	let baseProperty: String

	// user declares:
	memberwise init() {}
	// compiler synthesizes:
	init(baseProperty: String) {
		self.baseProperty = baseProperty
	}
}

class Derived: Base {
	let derivedProperty: Int

	// user declares:
	memberwise init() {}
	// compiler synthesizes (adding forwarded memberwise parameters):
	init(baseProperty: String, derivedProperty: Int) {
		self.derivedProperry = derivedProperty
		super.init(baseProperty: baseProperty)
	}
}

```

## Detailed design

### Syntax changes

This proposal introduces two new syntactic elements: the `memberwise` initializer declaration modifier and the `@nomemberwise` property attribute.

Initializers will be able to opt-in to synthesized memberwise initialization with the `memberwise` declaration modifier.  This modifier will cause the compiler to follow the procedure outlined later in the design to synthesize memberwise parameters as well as memberwise initialization code at the beginning of the initializer body.

Properties will be able to opt-out of memberwise initialization with the `@nomemberwise` attribute.  When they do so they will not be eligible for memberwise initialization synthesis.  Because of this they must be initialized directly with an initial value or initialized directly by every initializer for the type.

### Overview

Throughout this design the term **memberwise initialization parameter** is used to refer to initializer parameters synthesized by the compiler as part of memberwise initialization synthesis.

### Algorithm

The steps described in this section will be followed by the compiler when it performs memberwise initialization synthesis.  These steps supercede the synthesis of initialization for properties with initial values that exists today.

When the compiler performs memberwise initialization synthesis it will determine the set of properties that are eligible for synthesis that **are not** directly initialized in the body of the initializer.  It will then synthesize parameters for them as well the initialization of them at the beginning of the initializer body.

#### Terminology

* **direct memberwise initialization parameters** are parameters which are synthesized by the compiler and initialized directly in the body of the initializer.

* **forwarded memberwise initialization parameters** are parameters which are synthesized by the compiler and provided to another initializer that is called in the body of the initializer.

* **synthesized memberwise initialization parameters** or simply *memberwise initialization parameters* is the full set of parameters synthesized by the compiler which includes both direct and forwarded memberwise initialization parameters.

#### Designated initializers and non-delegating struct initializers

1. Determine the set of properties elibile for memberwise initialization synthesis.  This set is known as the set of *direct memberwise initialization parameters*.  In order to be eligible for memberwise initialization synthesis a property **must** be *at least* as visible as the initializer itself, **must not** have the `@nomemberwise` attribute, and **must not** have a behavior that does not allow memberwise initialization.  Currently `lazy` is an example of such a behavior that should prohibit memberwise initialization.  If both this proposal and the Property Behaviors proposal are accepted we will need a way for behaviors to specify whether they are compatible with memberwise initialization or not.

2. If any of the properties in that set produced in step one are directly initialized in the body of the initializer **or** have a name identical to an external parameter name for the intitializer remove them from the set.  If the initializer contains a parameter with an external label matching the name of a property that is eligible for memberwise intialization it must initialize that property directly.

3. When performing memberwise initialization for a subclass, inspect the call it makes to its superclass initialzier.  Determine the set of *synthesized memberwise initialization parameters* that exist for the superclass initializer that is called.  These parameters **may** participate in *memberwise initialization parameter forwarding*.  The set is known as the set of *forwarded memberwise initialization parameters*.

4. If the subclass initializer provides arguments for any of the parameters identified in step three remove them from the set.  Because a value is provided for them directly synthesized forwarding is not necessary.

5. If the superclass property corresponding to any of the remaining *forwarded memberwise initialization parameters* has a lower visibility than the initializer itself report a compilation error.  These parameters **must** be supplied directly by the subclass initializer.

6. Divide each of the sets gathered in step one and step three into two subsets, one of properties that contain initial values and the other containing properties that *do not* contain initial values.

7. Synthesize *memberwise initialization parameters* at the end of the initializer parameter list, but immediately prior to a trailing function parameter if such a parameter exists.  The synthesized parameters should have external labels matching the property name.  Place the synthesized parameters in the following order:
	1. *forwarded memberwise initialization parameters* that **do not** have an initial value in the same order they appear in the superclass initializer.
	2. *direct memberwise initialization parameters* that **do not** have an initial value in the order in which their corresponding properties are declared.
	3. *forwarded memberwise initialization parameters* that **do** have an initial value in the same order they appear in the superclass intitializer.  Also synthesize a default value matching the initial value for these parameters.
	4. *direct memberwise initialization parameters* that **do** have an initial value in the order in which their corresponding properties are declared.

8. Synthesize initialization of all *direct memberwise initialization parameters* at the beginning of the initializer body.

9. Synthesize the initialization of any properties which are ineligible for memberwise initialization, **are not** initialized elsewhere in the initializer body, and which **do have** an initial value provided in their declaration.  This step is identical to the synthesis of initialization for properties that declare initial values that happens today, but applies to a more restricted set of properties: those which **are not** initialized directly **and are not** eligible for memberwise initialization synthesis (when the initializer contains the `memberwise` declaration modifier).  

	ASIDE: it would be desirable to suppress the synthesis of properties that declare an initial value if that property is initialized directly in the body of the initializer whether or not the initializer opts-in to memberwise initialization.  This does not currently happen today, making it impossible to override an initial value for immutable properties with a different value in the body of an initializer.

10. Synthesize arguments to the superclass initializer for *forwarded memberwise initialization parameters*.  The call to the superclass initializer in the memberwise initializer body must be updated to forward any *forwarded memberwise initialization parameters* that were synthesized by the compiler.

#### Convenience and delegating initializers

Convenience initializers for classes and delegating initializers use the same algorithm for *forwarding memberwise initialization parameters* as described in the previous steps.  They do not include any *direct memberwise initialization parameters* and do not synthesize initialization of any stored properties in the body of the initializer.

### Objective-C Class Import

Objective-C frameworks are extremely important to (most) Swift developers.  In order to provide the call-site advantages of flexible memberwise initialization to Swift code using Cocoa frameworks this proposal recommends introducing a `MEMBERWISE` attribute that can be applied to Objective-C properties and initializers.

Mutable Objective-C properties can be marked with the `MEMBERWISE` attribute.  Readonly Objective-C properties cannot be marked with the `MEMBERWISE` attribute.  The `MEMBERWISE` attribute should only be used for properties that are initialized with a default value (not a value provided directly by the caller or computed in some way) in **all** of the class's initializers.

Objective-C initializers may also be marked with the `MEMBERWISE` attribute.  When Swift imports an Objectiv-C initializer marked with this attribute it allows callers to provide memberwise values for the properties declared in the class that are marked with the `MEMBERWISE` attribute.  At call sites for these initializers the compiler performs a transformation that results in the memberwise properties being set with the provided value immediately after initialization of the instance completes.

It may also be desirable to allow specific initializers to hide the memberwise parameter for specific properties if necessary.  `NO_MEMBERWISE(prop1, prop2)`

It is important to observe that the mechanism for performing memberwise initialization of Objective-C classes (post-initialization setter calls) is implemented in a different way than native Swift memberwise initialization.  As long as developers are careful in how they annotate Objective-C types this implementation difference should not result in any observable differences to callers.  

The difference in implementation is necessary if we wish to use call-site memberwise initialization syntax in Swift when initializing instances of Cocoa classes.  There have been several threads with ideas for better syntax for initializing members of Cocoa class instances.  I believe memberwise initialization is the *best* way to do this as it allows full configuration of the instance in the initializer call. 

Obviously supporting memberwise initialization with Cocoa classes would require Apple to add the `MEMBERWISE` attribute where appropriate.  If this proposal is accepted with the Objective-C class import provision intact my hope is that this will happen as it has in other cases where annotations are necessary to improve Swift interoperability.  If Apple does not intend to do so it may be desirable to remove the Objective-C interop portion of this proposal.

## Impact on existing code

The changes described in this proposal are strictly additive and will have no impact on existing code.

One possible breaking change which may be desirable to include alongside this proposed solution is to elimintate the existing memberwise initializer for structs and require developers to specifically opt-in to its synthesis by writing `memberwise init() {}`.  A mechanical transformation is possible to generate this declaration automatically if the existing memberwise initializer is removed.

## Alternatives considered

### Require stored properties to opt-in to memberwise initialization

This is a reasonable option and and I expect a healthy debate about which default is better.  The decision to require opt-out was made for several reasons:

1. The memberwise initializer for structs does not currently require an annotation for properties to opt-in.  Requiring an annotation for a mechanism designed to supercede that mechanism may be viewed as boilerplate.
2. Stored properties with public visibility are often intialized directly with a value provided by the caller.
3. Stored properties with **less visibility** than a memberwise initializer are not eligible for memberwise initialization.  No annotation is required to indicate that.

I do think a strong argument can be made that it may be **safer** and **more clear** to require an `@memberwise` attribute on stored properties in order to opt-in to memberwise initialization.  I am very interested in community input on this.

### Allow all initializers to participate in memberwise initialization

This option was not seriously considered.  It would impact existing code and it would provide no indication in the declaration of the initializer that the compiler will synthesize additional parameters and perform additional initialization of stored properties in the body of the initializer.

### Require initializers to opt-out of memberwise initialization

This option was also not seriously considered.  It has the same problems as allowing all initializers to participate in memberwise initialization.

### Require initializers to explicitly specify memberwise initialization parameters

The thread "[helpers for initializing properties of the same name as parameters](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151130/000428.html)" discussed an idea for synthesizing property initialization in the body of the initializer while requiring the parameters to be declard explicitly.  

```swift
struct Foo {
    let bar: String
    let bas: Int
    let baz: Double
    init(self.bar: String, self.bas: Int, bax: Int) {
		  // self.bar = bar synthesized by the compiler
		  // self.bas = bas synthesized by the compiler
        self.baz = Double(bax)
    }
}

```

The downside of this approach is that the boilerplate parameter declarations grow at the rate MxN (properties x initializers).  It also does not address forwarding of memberwise initialization parameters which makes it useless for convenience and delegating initializers.

Proponents of this approach believe it provides additional clarity and control over the current proposal.  

Under the current proposal full control is still available.  It requires initializers to opt-in to memberwise initialization.  The boilerplate saved in the examples on the list is relatively minimal and is tolerable in situations where full control of initialization is required.

I believe the `memberwise` declaration modifier on the initializer makes it clear that the compiler will synthesize additional parameters.  Furthermore, IDEs and generated documentation will contain the full, synthesized signature of the initializer.  
