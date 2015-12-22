# Flexible Memberwise Initialization

* Proposal: [SE-NNNN](https://github.com/apple/swift-evolution/blob/master/proposals/NNNN-flexible-memberwise-initializers.md)
* Author(s): [Matthew Johnson](https://github.com/anandabits)
* Status: **Review**
* Review manager: TBD

## Introduction

The Swift compiler is currently able to generate a memberwise initializer for us in some circumstances however there are currently many limitations to this.  This proposal build on the idea of compiler generated memberwise initialization making it available to any initializer that opts in.

Swift-evolution thread: [Proposal Draft: flexible memberwise initialization](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151221/003902.html)

## Motivation

When designing initializers for a type we are currently faced with the unfortunate fact that the more flexibility we wish to offer users the more boilerplate we are required to write and maintain.  We usually end up with more boilerplate and less flexibility than desired.  There have been various strategies employed to mitigate this problem.  

Sometimes properties that should be immutable are made mutable and a potentially unsafe ad-hoc two-phase initialization pattern is employed where an instance is initialized and then configured immediately afterwards.  When properties that need to be mutable have a sensible default value they are simply default-initialized and the same post-initialization configuration strategy is employed when the default value is not correct for the intended use.  This results in an instance which may pass through several incorrect states before it is correctly initialized for its intended use.

Underlying this problem is the fact initialization scales with M x N complexity (M members, N initializers).  We need as much help from the compiler as we can get!

Flexible and concise initialization for both type authors and consumers will encourages using immutability where possible and removes the need for boilerplate from the concerns one must consider when designing the intializers for a type.

Quoting [Chris Lattner](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20151130/000518.html):

	The default memberwise initializer behavior of Swift has at least these deficiencies (IMO):
	1) Defining a custom init in a struct disables the memberwise initializer, and there is no easy way to get it back.
	2) Access control + the memberwise init often requires you to implement it yourself.
	3) We don’t get memberwise inits for classes.
	4) var properties with default initializers should have their parameter to the synthesized initializer defaulted.
	5) lazy properties with memberwise initializers have problems (the memberwise init eagerly touches it).

Add to the list “all or nothing”.  The compiler generates the entire initializer and does not help to eliminate boilerplate for any other initializers where it may be desirable to use memberwise intialization for a subset of members and initialize others manually.

It is common to have a type with a number of public members that are intended to be configured by clients, but also with some private state comprising implementation details of the type.  This is especially prevalent in UI code which may expose many properties for configuring visual appearance, etc.  Flexibile memberwise initialization can provide great benefit in these use cases, but it immediately becomes useless if it is "all or nothing".  

We need a flexible solution that can synthesize memberwise initialization for some members while allowing the type auther full control over initialization of implementation details.

## Proposed solution

I propose adding a `memberwise` declaration modifier for initializers which allows them to *opt-in* to synthesis of memberwise initialization and a `@nomemberwise` attribute allowing properties to *opt-out* of such synthesis and memberwise initializers to explicitly prevent memberwise initialization of specific properties. 

The programmer model is as straighforward as possible.  

The set of properties that receive memberwise initialization parameters is determined by considering *only* the initializer declaration and the declarations for all properties that are *at least* as visible as the initializer (including any behaviors attached to the properties).  The rules are as follows:

1. Their access level is *at least* as visible as the memberwise initializer.
2. They do not have a behavior which prohibits memberwise initialization.
3. The property **is not** annotated with the `@nomemberwise` attribute.
4. The property **is not** included in the `@nomemberwise` attribute list attached of the initializer.  If `super` is included in the `@nomemberwise`

The parameters are synthesized in the parameter list in the location of the `...` placeholder.  They are ordered as follows:

1. All properties **without** default values precede properties **with** default values.
2. Within each group, superclass properties precede subclass properties.
3. Finally, follow declaration order

This section of the document contains several examples of the solution in action.  It *does not* cover every possible scenario.  If there are concrete examples you are wondering about please post them to the list.  I will be happy to discuss them and will add any examples we consider important to this section as the discussion progresses.

Specific details on how synthesis is performed are contained in the detailed design.

### Replacing the current memberwise initializer

```swift
struct S {
	let s: String
	let i: Int

	// user declares:
	memberwise init(..) {}
	// compiler synthesizes:
	init(s: String, i: Int) {
		/* synthesized */ self.s = s
		/* synthesized */ self.i = i
	}
}
```

### Properties with initial values

NOTE: this example is not valid under the current initialization rules.  I feel strongly that this example *should* be possible and discuss it further in the detailed design section.  The example can be made valid under the current initialization rules if the properties are changed from `let` to `var`.

```swift
struct S {
	let s: String = "hello"
	let i: Int = 42

	// user declares:
	memberwise init(...) {}
	// compiler synthesizes:
	init(s: String = "hello", i: Int = 42) {
		/* synthesized */ self.s = s
		/* synthesized */ self.i = i
	}
}
```

If the initialization rules *are not* modified it would be possible to support this use case with an additional attribute:

```swift
struct S {
	@default("hello") let s: String
	@default(42) let i: Int

	// user declares:
	memberwise init(...) {}
	// compiler synthesizes:
	init(s: String = "hello", i: Int = 42) {
		/* synthesized */ self.s = s
		/* synthesized */ self.i = i
	}
}
```

This is not a great option, but it is better than requiring manual initialization when this is the correct implementation.

### Partial memberwise initialization

NOTE: The example in this secion doesn't really save a lot.  Imagine ten properties with only one excluded from memberwise initialization.

```swift
struct S {
	let s: String
	let i: Int

	// user declares:
	@nomemberwise(i)
	memberwise init(...) {
		i = getTheValueForI()
	}
	// compiler synthesizes (suppressing memberwise initialization for properties mentioned in the @nomemberwise attribute):
	init(s: String) {
		/* synthesized */ self.s = s
		
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
	memberwise init(...) {
		// compiler error, i memberwise initialization cannot be synthesized 
		// for i because it is less visible than the initializer itself
	}
}
```

```swift
struct S {
	let s: String
	private let i: Int

	// user declares:
	memberwise init(...) {
		i = 42
	}
	// compiler synthesizes (suppressing memberwise initialization for properties with lower visibility):
	init(s: String) {
		/* synthesized */ self.s = s
		
		// body of the user's initializer remains
		i = 42
	}
}
```

### lazy properties and incompatible behaviors

```swift
struct S {
	let s: String
	lazy var i: Int = InitialValueForI()

	// user declares:
	memberwise init(...) {
	}
	// compiler synthesizes:
	init(s: String) {
		/* synthesized */ self.s = s
		
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
	memberwise init() {
		// compiler error, i is not initialized
	}
}
```

```swift
struct S {
	let s: String
	@nomemberwise let i: Int

	// user declares:
	memberwise init() {
		i = 42
	}
	// compiler synthesizes:
	memberwise init(s: String) {
		/* synthesized */ self.s = s
		
		// body of the user's intializer remains
		i = 42
	}
}
```

```swift
struct S {
	let s: String
	@nomemberwise let i: Int

	// user declares:
	memberwise init(configuration: SomeTypeWithAnIntMember, ...) {
		i = configuration.intMember
	}
	// compiler synthesizes (suppressing memberwise intialization for properties with the @nomemberwise attribute):
	init(configuration: SomeTypeWithAnIntMember, s: String) {
		/* synthesized */ self.s = s
		
		// body of the user's initializer remains
		i = configuration.intMember
	}
}
```


### delegating and convenience initializers

NOTE: the call to the chained initializer **must** be unambiguous when only the explitly provided arguments are considered.  However it is allowed to be invalid on its own as long as it becomes valid after forwarding of memberwise arguments is completed (i.e. forwarding can be used to supply memberwise arguments where the parameter *does not* have a default value).

```swift

struct S {
	let s: String = "hello"
	let i: Int = 42

	// user declares:
	memberwise init(...) {}
	// compiler synthesizes:
	init(s: String = "hello", i: Int = 42) {
		self.s = s
		self.i = i
	}
	
	// user declares:
	@nomemberwise(s)
	memberwise init(describable: CustomStringConvertible, ...) {
		self.init(s: describable.description)
	}
	// compiler synthesizes (adding forwarded memberwise parameters):
	init(describable: CustomStringConvertible, i: Int = 42) {
		self.init(s: describable.description, i: i)
	}
}
```

### subclass initializers

NOTE: the call to the chained initializer **must** be unambiguous when only the explitly provided arguments are considered.  However it is allowed to be invalid on its own as long as it becomes valid after forwarding of memberwise arguments is completed (i.e. forwarding can be used to supply memberwise arguments where the parameter *does not* have a default value).

```swift
class Base {
	let baseProperty: String

	// user declares:
	memberwise init(...) {}
	// compiler synthesizes:
	init(baseProperty: String) {
		self.baseProperty = baseProperty
		super.init()
	}
}

class Derived: Base {
	let derivedProperty: Int

	// user declares:
	memberwise init(...) {}
	// compiler synthesizes (adding forwarded memberwise parameters):
	init(baseProperty: String, derivedProperty: Int) {
		self.derivedProperry = derivedProperty
		super.init(baseProperty: baseProperty)
	}
}

```

## Detailed design

### Syntax changes

This proposal introduces three new syntactic elements: the `memberwise` initializer declaration modifier, the `...` memberwise parameter placeholder, and the `@nomemberwise` attribute which can be applied to both properties and initializers.

Initializers opt-in to synthesized memberwise initialization with the `memberwise` declaration modifier.  This modifier will cause the compiler to follow the procedure outlined later in the design to synthesize memberwise parameters as well as memberwise initialization code at the beginning of the initializer body.  Memberwise initializers can explicitly prevent memberwise initialization for specific properties by including them in a list of property names provided to the `@nomemberwise` attribute like this: `@nomemberwise(prop1, prop2)`.

Properties will be able to opt-out of memberwise initialization with the `@nomemberwise` attribute.  When they do so they will not be eligible for memberwise initialization synthesis.  Because of this they must be initialized directly with an initial value or initialized directly by every initializer for the type.

### Overview

Throughout this design the term **memberwise initialization parameter** is used to refer to initializer parameters synthesized by the compiler as part of **memberwise initialization synthesis**.

#### Terminology

* **direct memberwise initialization parameters** are parameters which are synthesized by the compiler and initialized directly in the body of the initializer.

* **forwarded memberwise initialization parameters** are parameters which are synthesized by the compiler and provided to another initializer that is called in the body of the initializer.

* **synthesized memberwise initialization parameters** or simply *memberwise initialization parameters* is the full set of parameters synthesized by the compiler which includes both direct and forwarded memberwise initialization parameters.

#### Memberwise initializer chaining

Convenience initializers and delegating memberwise initializers **must** call an initializer that is also a memberwise initializer in order to allow the memberwise arguments to be forwarded.  

Memberwise designated initializers in subclasses **must** *either* call a memberwise initializer in the superclass *or* include `super` in its `@nomemberwise` attribute list as follows: `@nomemberwise(super)`.

When calling an initializer with lower visibility arguments **must** be provided directly in the initializer body for any memberwise initialization parameters corresponding to properties with lower visibility than the *calling* initializer.  This is because the *calling* initializer is not able to synthesize and forward memberwise initialization parameters for these properties as that would violate the first property eligibility rule.

#### Algorithm

1. Determine the set of properties elibile for memberwise initialization synthesis.  All properties, including inherited properties are considered.  Properties are eligible for memberwise initialization synthesis if:

	1. Their access level is *at least* as visible as the memberwise initializer.
	2. They do not have a behavior which prohibits memberwise initialization.
	3. The property **is not** annotated with the `@nomemberwise` attribute.
	4. The property **is not** included in the `@nomemberwise` attribute list attached of the initializer.  If `super` is included in the `@nomemberwise` attribute list all inherited properties are ineligible for memberwise initialization.
	
	The current initialization rules also require any `let` properties with an initial value to be excluded from memberwise initialization.  I believe it is extremely desirable to allow memberwise initialization for `let` properties while still supplying a default value in the property declaration.  If this is not allowed we have three choices, none of which are great: 
	
	1. Make the property a `var`.
	2. Do not supply a default value and require callers to explicitly provide a value at every call site.
	3. Manually declare a parameter with the default value in every initializer and manually initialize the property with the supplied argument.
	
	I would also like to point out that *instance* `let` properties with initial *values* really do not make sense at all if initializers are not able to specify a value other than the default.  The result is that **all** instances are required to have the same value which wastes space.
	
2. If the initializer declares any parameters with external labels matching the name of any of the properties eligible for memberwise initialization report a compiler error.  If the initializer needs to declare a parameter with an external label matching the name of a property the property **must** be explicitly excluded from memberwise initialization, possibly via the `@nomemberwise` attribute of the initializer (lower visibility is another possibility).

3. If the initialzer is a delegating, convenience, or subclass designated initializer and it needs to explicitly provide an argument for a memberwise parameter when it calls the chained initializer it must ensure that the corresponding property is excluded from its memberwise synthesis set, possibly via its `@nomemberwise` attribute (lower visibility is another possibility).  If it *does* provide an explicit argument for a memberwise parameter included in its synthesis set report a compiler error.  

4. Synthesize *memberwise initialization parameters* in the location where the `...` placeholder was specified.  The synthesized parameters should have external labels matching the property name.  Place the synthesized parameters in the following order:
	1. All properties **without** default values precede properties **with** default values.
	2. Within each group, superclass properties precede subclass properties.
	3. Finally, follow declaration order
	
5. Synthesize initialization of all *direct memberwise initialization parameters* at the beginning of the initializer body.

6. Synthesize arguments to the superclass initializer for *forwarded memberwise initialization parameters*.  The call to a chained initializer in the memberwise initializer body must be updated to forward any *forwarded memberwise initialization parameters* that were synthesized by the compiler.

7. If the initialization rules are modified to allow initializers to initialize a property (especially a `let` property) with a value *other than* its initial value, synthesis of initialization with the default / initial value would happen *after* the preceding steps have been completed.  This would allow memberwise initialization to take control of initialization of such properties where appropriate, while still allowing default / initial values initialization to be synthesized for members which do not participate in memberwise initialization and which have not been explicityl intialized in the body of the initializer.  

	One possible implementation would be to attempt compilation and if a 'Return from initializer without initializing all stored properties' error is generated, attempt recovery by synthesizing initialization of the property if an initial value was provided in the declaration.  This is probably not the best implementation, but I mention it as it demonstrates that the compiler is already able to detect *when* it would be necessary.

## Impact on existing code

The changes described in this proposal are strictly additive and will have no impact on existing code.

One possible breaking change which may be desirable to include alongside this proposed solution is to elimintate the existing memberwise initializer for structs and require developers to specifically opt-in to its synthesis by writing `memberwise init(...) {}`.  A mechanical transformation is possible to generate this declaration automatically if the existing memberwise initializer is removed.

## Future enhancements

### Objective-C Class Import

Objective-C frameworks are extremely important to (most) Swift developers.  In order to provide the call-site advantages of flexible memberwise initialization to Swift code using Cocoa frameworks a future proposal could recommend introducing a `MEMBERWISE` attribute that can be applied to Objective-C properties and initializers.

Mutable Objective-C properties could be marked with the `MEMBERWISE` attribute.  Readonly Objective-C properties **could not** be marked with the `MEMBERWISE` attribute.  The `MEMBERWISE` attribute should only be used for properties that are initialized with a default value (not a value provided directly by the caller or computed in some way) in **all** of the class's initializers.

Objective-C initializers could also be marked with the `MEMBERWISE` attribute.  When Swift imports an Objective-C initializer marked with this attribute it could allow callers to provide memberwise values for the properties declared in the class that are marked with the `MEMBERWISE` attribute.  At call sites for these initializers the compiler could perform a transformation that results in the memberwise properties being set with the provided value immediately after initialization of the instance completes.

It may also be desirable to allow specific initializers to hide the memberwise parameter for specific properties if necessary.  `NOMEMBERWISE(prop1, prop2)`

It is important to observe that the mechanism for performing memberwise initialization of Objective-C classes (post-initialization setter calls) must be implemented in a different way than native Swift memberwise initialization.  As long as developers are careful in how they annotate Objective-C types this implementation difference should not result in any observable differences to callers.  

The difference in implementation is necessary if we wish to use call-site memberwise initialization syntax in Swift when initializing instances of Cocoa classes.  There have been several threads with ideas for better syntax for initializing members of Cocoa class instances.  I believe memberwise initialization is the *best* way to do this as it allows full configuration of the instance in the initializer call. 

Obviously supporting memberwise initialization with Cocoa classes would require Apple to add the `MEMBERWISE` attribute where appropriate.  A proposal for the Objective-C class import provision is of significantly less value if this did not happen.  My recommendation is that an Objective-C import proposal should be drafted and submitted if this proposal is submitted, but not until the core team is confident that Apple will add the necessary annotations to their frameworks.

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

### Allow parameters to be synthesized for properties with a lower access level than the initializer

I considered allowing parameters to be synthesized for properties that are not directly visible to callers of the initializer.  There is no direct conflict with the access modifier and it is possible to write such code manually.  I decided against this approach because as it is unlikely to be the right approach most of the time.  In cases where it is the right approach I think it is a good thing to require developers to write this code manually.

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

The downside of this approach is that it does not address the M x N scaling issue mentioned in the motivation section.  The manual initialization statements are elided, but the boilerplate parameter declarations still grow at the rate MxN (properties x initializers).  It also does not address forwarding of memberwise initialization parameters which makes it useless for convenience and delegating initializers.

Proponents of this approach believe it provides additional clarity and control over the current proposal.  

Under the current proposal full control is still available.  It requires initializers to opt-in to memberwise initialization.  When full control is necessary an initializer will simply not opt-in to memberwise initialization synthesis.  The boilerplate saved in the examples on the list is relatively minimal and is tolerable in situations where full control of initialization is required.

I believe the `memberwise` declaration modifier on the initializer and the placeholder in the parameter list make it clear that the compiler will synthesize additional parameters.  Furthermore, IDEs and generated documentation will contain the full, synthesized signature of the initializer.  

Finally, this idea is not mutually exclusive with the current proposal.  It could even work in the declaration of a memberwise initializer, so long the corresponding property was made ineligible for memberwise intialization synthesis.
