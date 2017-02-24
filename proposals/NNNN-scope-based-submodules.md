# Scope-based submodules

* Proposal: [SE-NNNN](NNNN-scope-based-submodules.md)
* Authors: [Matthew Johnson](https://github.com/anandabits)
* Review Manager: TBD
* Status: **Awaiting review**


## Introduction

This proposal describes a submodule system based on the principle of strictly nested scopes.  The design strives to follow the Swift philosophy of offering good defaults while progressively disclosing very powerful tools that can be used by experts to solve complex problems.


## Motivation

Swift currently provides two kinds of entities that provide system boundaries without directly introducing a symbol: modules and files*.

* Modules introduce an ABI boundary, a name boundary, and a scope boundary.
* Files introduce a scope boundary that carries no additional semantics.

Swift currently lacks the ability to introduce a name and scope boundary without also introducing an ABI boundary.  Such a boundary would be naturally situated halfway between in terms of strength and ceremony.  The lack of such a boundary significantly limits our abiltiy to structure a large Swift program.  Introducing a way to form this kind of boundary will provide a powerful tool for giving internal structure to a module.

*The important aspect of a file in Swift is the logical scope boundary it introduces.  The physical file system representation is incidental to this.  The appendix on file system independence discusses this in more detail.


## Goals

Any discussion of submodules inevitably reveals that there are very different perspectives of what a submodule is, what problems submodules should be able to solve, etc.  This section describes the design goals of this proposal in order to facilitate evaluation of both the goals themselves as well as how well the solution accomplishes those goals.

The primary goal of this proposal are to introduce a unit of encapsulation within a module that is larger than a file as a means of adding explicit structure to a large program.  All other goals are subordinate to this goal and should be considered in light of it.  

Some other goals of this proposal are:

* Submodules should help us to manage and understand the internal dependencies of a large, complex system.
* Submodules should be able to collaborate with peer submodules without necessarily being exposed to the rest of the module.
* A module should not be *required* to expose its internal submodule structure to users when symbols are exported.
* It should be possible to extract a submodule from existing code with minimal friction.  The only difficulty should be breaking any circular dependencies.

Some additional non-functional requirements for the solution are:

* Submodules should not negatively impact runtime performance.  WMO should be able to see across submodule boundaries.
* Submodules should not negatively impact build performance.  Ideally they will improve build performance by giving the compiler more high-level information about internal dependencies.

Deferred goal:

* It is not an immediate goal to support submodules in single-file scripts.  The appendix discussing file system independence discusses some ideas that could be used to support single-file scripts in the future.



## Proposed solution

There are several relatively orthogonal aspects to the design of a submodule system.  A design must answer the following questions:

* How is code placed in a submodule?
* How are symbols in one submodule made available to another submodule or module?
* How do submodules interact with access control?

This proposal diverages a little bit from the usual proposal format to faciliate discussion of alternatives within the context of each aspect of the chosen design.  In each case an alternative could be substituted without compromising the overall design.

**Note:** This proposal uses the term "top-level submodule" to distinguish the scope of code that is not explicitly placed in a submodule from the module as a whole.  The top-level submodule is in most respects identical to any other parent submodule.  There are two differences: 1) it is the only submodule that can export symbols outside of the module and 2) any `open` or `public` symbols it declares are automatically exported according to their access modifier.


### Placing code in a submodule

Each file is part of the top level submodule by default.

##### A `submodule` declaration may be used at the top of a file:

* Only the first non-comment, non-whitespace line may contain a submodule declaration.
* The submodule decalaration looks like this: `submodule MySubmoduleName`
* A submodule declaration may not be prefixed with the module name.

##### Submodule names form a hierarchical path:
* The fully qualified name of the submodule specified by `Submodule.InnerSubmodule` is: `MyModuleName.Submodule.InnerSubmodule`.
* In this example, `InnerSubmodule` is a child of `Submodule`.
* A submodule may not have the same name as any of its ancestors.  This follows the rule used by types.

##### Submodules may not be extended.  They form strictly nested scopes.
* The *only* way to place code in a submodule is with a `submodule` declaration at the top of a file.
* All code in a file exists in a single submodule. 

A module is made up of strictly nested scoped that look like this:

![The hierarchy of nested scopes in scope-based submodules](NNNN-scope-based-submodules.png "The hierarchy of nested scopes in scope-based submodules")

#### Alternatives

##### Grouping mechanisms

There are several other ways to specify which submodule the top-level scope of a file is in.  All of these alternatives share a crucial problem: you can't tell what submodule your code is in by looking at the file.  

The alternatives are:

* Use a manifest file. This would be painful to maintain.
* Use file system paths.  This is too tightly coupled to physical organization.  Appendix A discusses file system independence in more detail. 
* Leave this up to the build system.  This makes it more difficult for a module to support multiple build systems.


##### Require all files to include a `submodule` declaration

We could require all files to include an explicit submodule declaration.  However, that would be a breaking change and would violate the principle of progressive disclosure.  Users should not need to know about submodules if they don't use them.

##### Allow submodule references to explicitly state the name of the module

The module name is implicit throughout the scope of the entire module.  Specifying it explicitly is redundant.  Prohibiting explicit mention of the module name offers more flexibility for combining submodules into build products.


### Visibility of submodules

The `export` statement is used to modifiy the visibiltiy of a submodule within the module.  It is also used by the top-level module to publish submodules to clients of the module.

* All submodules are implicitly exported with module-wide visibility by default (and hidden outside of the module by default*).
* All submodules are implicitly *available* for export outside the module.
* A submodule may use an explicit `export` statement to *modify* the visibility of a descendent submodule.
* `export` statements are only allowed at the top level of a file. 

*The exception to this is that `open` and `public` symbols in the top-level submodule are always exported exactly as declared.

#### Top-level export

All export statements consist of an access modifier, the `export` keyword, and a submodule name:

`open export ChildSubmodule`

When this export statement appears in the top-level submodule, `ChildSubmodule` becomes available for import by clients of the submodule with the fully qualified name `Module.ChildSubmodule`.  `public` exports are also available.  When `public` is used all published symbols in the exported submodule have a maximum visiblity of `public` regardless of how they were declared.

Top-level `public` and `open` export statement may be modified with the following options:

* A submodule may be published under a different external name using the `export as NewName` syntax*.
* `@implicit` causes symbols from the submodule to be implicitly imported when the module is imported.
* `@inline` causes the symbols from the submodule to appear as if they had been declared directly within the top-level submodule.
* `@implicit` may be combined with renaming but `@inline` may not appear along with either of them.

*When a submodule is renamed for export with the `as` clause its internal name does not change.  A submodule always has the same fully qualfied name everywhere within its module.

Here are some example export statements:

```swift
// All symbols in `Child1` are available for import by clients as `Module.Foo`
// These symbols are *not* imported automatically when a client imports the module with `import Module`
public export Child1 as Foo

// All symbols in `Child2` are available for explicit import by clients as `Module.Child2`
// The symbols are also automatically imported when a client imports the module with `import Module`
// When the symbols are imported implicitly they retain the fully qualified name prefix of `Module.Child2`
@implicit open export Child2

// All symbols in `Child3` are available for explicit import by clients as `Module.Foo.Bar`
// The symbols are also automatically imported when a client imports the module with `import Module`
// When the symbols are imported implicitly they retain the fully qualified name prefix of `Module.Foo.Bar`
@implicit open export Child3 as Foo.Bar

// All symbols in `Child4.Grandchild` appear to clients as if they had been declared
// directly in the top-level submodule.
// If the process of inlining the symbols produces duplicate symbols a compiler error is produced
// at the site of one or both of the `export` statements.
@inline public export Child4.Grandchild

// All symbols in `Child5.Grandchild` are available for explicit import by clients as `Module.Foo` 
// along with the symbols declared in `Child`.
// The symbols are also automatically imported when a client imports the module with `import Module`
// When the symbols are imported implicitly they retain the fully qualified name prefix of `Module.Foo`
// As with `@inline`, when two submodules are given the same external name a duplicate symbol error may occur.
@implicit public export Child5.Grandchild as Foo
```

One interesting observation is that both `Child1` and `Child5.Grandchild` are renamed to `Foo`.  The symbols declared former is not implicitly imported by `import Module` but the latter is, despite having the same fully qualified name prefix.

#### Exports within the module

A submodule may bound the maximum visibility of any of its descendent submodules by explicitly exporting it:

```swift
// `Child1.Grandchild` may be exported by the top-level module, but only with `public` visibility.
// `Child1.Grandchild` may not be exported as `open`.
public export Child1.Grandchild

// `Child2` is exported with `internal` visibility.
// Because `internal` is scoped to the submodule level *only* the parent submodule can see `Child2`.
// No submodules except the direct parent of `Child2` (the current submodule) are allowed to `import Child2`.
// This also implies that the `Child2` may not be exported to clients because the top-level 
// submodule is not able to see or reference `Child2` at all.
internal export Child2
```

* The access modifier may specify a scope `internal` or greater.  
* Only the **direct** parent of a submodule may specify the `internal` modifier.  A grandparent cannot hide a grandchild from its parent.
* If a descendent includes an `export` statement for the same submodule, the access modifier must be **no greater than** the access modifier specified by the descendent.  An ancestor may provide a tighter bound to visibility but may not increase visibility.  An attempt to increase visibility results in an error.

**Note:** If a submodule is not visible none of its descendents is visible either.

#### Alternatives

##### Use `import` access modifiers to export a submodule

The semantics of placing a bound on the visibility of a descendent submodule is significantly different than the semantics of importing symbols from a submodule into the current lexical scope.  Mixing the semantics of the two is confusing.

##### Restrict the visibility of a submodule to its parent unless the parent explicitly exports it.

Users should be able to use submodules without needing to place `export` statements in every parent submodule.  Module-wide default visibility for submodules is analagous to `internal` default visibility for symbols.

##### Require all submodules to be visible module-wide.

This removes an important tool for bounded collaboration within a complex system.  A parent submodule should be allowed to have a child submodule(s) which are implementation details of the parent and not exposed to the rest of the module.

##### Allow renaming to be used by export statements within the module.

A submodule should have the same fully qualified name everywhere it is used within a single module, whether that be the declaring module or a client module.  The declaring module and client modules may see different names, but each sees a name that is consistent fully qualified name everywhere the submodule is referenced.

##### Allow `@inline` to be used by export statements within the module.

As with renaming, a symbol should have a single fully qualified name everywhere within a single module.

##### Allow `@implicit` to be used by export statements within the module.

This would reduce the visibility of internal dependencies.  If we find that import-per-submodule becomes boilerplate-y this is an easy feature to add later.

### Importing submodules

Submodules are imported in exactly the same way as an external module by using an `import` statement.  There are a few additional details that are not applicable for external modules:

* Circular imports are not allowed.
* A submodule may not import any of its ancestors.
* Relative child and sibling names are allowed using the same rules that apply to nested types.

### Access control

An access modifier applies an upper bound to the scope in which a symbol or submodule is visible.  With the introduction of submodules, `internal` now applies at the level of a submodule: only the current submodule may see an `internal` entity.  We need a new way to specify module-wide visibility.

This proposal builds on Option 2 in the proposal Fix Private Access Levels which reverts `private` to the Swift 2 meaning (equivalent to `fileprivate`) and uses `scoped` for the Swift 3 scoped access feature.  It does this by allowing the `scoped` access modifier to be parameterized with a scope reference.  By **defaults** it references the scope in which it appears, but any ancestor scope may be specified as a parameter.

The paremeterization of the `scoped` access modifier provides a simple yet powerful way for a submodule to bound the visibility of a descendent.

Some examples of using `scoped` exports are:

```swift
submodule Parent

// `Grandparent` and all of its descendents can see `Child1` (fully qualified: `Grandparent.Parent.Child1`)
// This reads: `Child1` is scoped to `Grandparent`.
scoped(Grandparent) export Child1

// `Child2` is visible throughout the module but may not be exported for use by clients.
// This reads: `Child2` is scoped to the module.
scoped(module) export Child2
```

With parameterization, `scoped` has the power to specify **all** access levels that Swift has today:

```
`scoped`                                      == `private` (Swift 3)
`scoped(file)`                                == `private` (Swift 2 & 4?) == `fileprivate` (Swift 3)
`scoped(submodule)`                           == `internal`
`scoped(public) scoped(internal, inherit)`*   == `public`
`scoped(public)`                              == `open`
```

This design is a direct generalization of the principle underlying Swift's existing access control system.  It unifies the semantics of the system under the single elegant mechanism of ancestor scope references.  

While it is *possible* to specify all access levels using `scoped` that is not recommended.  The aliases `public`, `private` (Swift 2) and `internal` provide excellent default access levels that don't require a user to think about scope hierarchies.  Using the default access levels when possible calls extra attention to cases where a different choice was made.

*This is a conceptual model.  This proposal does not introduce the `inherit` or `override` parameter to access modifiers.  It could be added in the future as a way to bound inheritance within a module.  It would work similarly to `private(set)` does in Swift today.

##### Aside

The parameterization of `scoped` also allows us to reference other scopes that we cannot in today's system, specifically extensions: `scoped(extension)` and outer types: `scoped(TypeName)`.

#### Alternatives

If we don't adopt the approach of parameterizing `scoped` our options for access control include:

##### Submodules are only allowed to see `public` and `open` symbols from other submodules

A module-wide scope is highly desirable.  People might avoid using submodules if this is not available.

This approach also creates a lot more friction when refactoring.  A possible workaround to the lack of a module-wide scope in this system is to place code in a non-exported submodule and declare symbols `public`.  Even with the workaround, extracting a submodule may not always be possible or desirable and the `public` access modifiers required would be misleading.  It would be much better to be able to state our intent directly.

##### Use `internal to cover the whole module and` `private` to cover a submodule

One suggestion that has appeared is the idea of removing `fileprivate` and making `private` be submodule-wide.  `internal` would remain module-wide.  This is too coarse - *many* people want a file-level scope.

`internal` is Swift's default access modifier.  A symbol with default access modifier should not be able to cross a submodule boundary implicitly.

##### Add the `moduleinternal` access modifier

This is about as ugly as `fileprivate`.



## Detailed design

### Export errors

#### Multiple exports of the same submodule

If a submodule exports the same descendent more than once and the semantics of the declarations are not identical an error is produced.

#### Symbol flattening

When a submodule is exported by the top-level module using the `@inline` attribute it is possible that there will be conflicting symbol definitions in the child and the top-level submodule (or other inlined submodules).  This results in a compiler error at the site of the conflicing `@inline export` statements.

#### Overlapping renames

As with flattening, when two or more submodules are given the same external name symbol conflicts are possible.  This also results in a compiler error at the site of the conflicting `export as` statements.

#### Access errors during export if the specified access modifier exceeds maximum

An error is produced when an export statement includes an access modifier greater than the bound provided for the exported submodule by a descendent of the exporting submodule.


## Source compatibility

This proposal is purely additive.  That said, it would be a breaking change for the standard library to move existing API into an externally visible submodule.

## Effect on ABI stability

This proposal is purely additive.  That said, it would be a breaking change for the standard library to move existing API into an externally visible submodule.

## Effect on API resilience

This proposal is purely additive.

## Future directions

### Selective export and import

The ability to import and export individual symbols would be a very nice to have.

### Scoped import

The ability to import modules, submodules, and symbols into any lexical scope would be nice to have.

### Bounded inheritance

It could be useful to have the ability to bound inheritance within a module.  This could be accomplished by inroducing `inherit` and `override` parameters for access modifiers (which would work similarly to the existing `set` parameter).


## Appendix A: file system independence

The submodule design specified by this proposal is file system independent.  The only relationship it has with the physical file system is that a file introduces an anonymous scope boundary which is referenced by `scoped(file)` or `fileprivate` (Swift 3) or `private` (Swift 2 and 4?).

The logical role of a "file" in this design is to provide a boundary of encapsulation that is even lighter weight than a submodule: it doesn't hide names.  All declarations are implicitly available not only *within* the file but also *across* the file boundary (modulo access control).  Files are to submodules as submodules are to modules.

If a future version of Swift were to eliminate files in favor of some kind of code browser it would still be very useful to have the ability to form a pure scope boundary (with no additional semantics).  A `scope` declaration could be used to do this.  Scope declarations could have an optional (or required) name and could even be nested.  In this system `private` would reference the nearest anonymous ancestor scope declaration (or would be removed if we don't allow anonymous scopes).

The logical structure of this design can be directly translated into a grammar that could be represented directly with syntax.  Such a grammer could be used to support scripts with submodules.  An example follows:

```
// A module contans a single implicit, anonymous submodule.
// submodule {
  // A submodule may contain `scope` declarations (i.e. files) as well as other submodules.
  // An anonymous scope is equivalent to a file in current Swift.
  // If we introduce lexical scopes we would probably require them to be named explicitly.
  // This example uses the anonymous scope in order to most closely match the role files play in the current system.
  // Because `scope` does not provide a name boundary all names declared in one scope
  // are visible in other scopes (modulo access control)
  scope {
      // Top-level declarations go here.
      // This is equivalent to the top level of a file in Swift today.
      // It is also equivalent to the top level of a file that does not contain 
      // a `submodule` declaration in this proposal.
      
      // It would be possible to allow nested, named scopes.
      // A scope name participates in the scope and name hierachies.
      // However, it does not form a name boundary like a submodule does.
      scope Named {
        // This declares the static variable `Named.foo`
        // `scoped(file)` references the nearest anonymous ancestor scope.
        // It is used in this example for specificity.
        // Real code would use the alias `private` or `fileprivate`
        // If we introduce explicit scope syntax we would probably want a better name to refer
        // to the nearest anonymous scope than `file` or we may just require all scopes to have a name.
        scoped(file) var foo: String
      }
      // `Named.foo` is visible here
  }
  // `Named.foo` is not visible here.
  
  submodule Foo {}
  submodule Baz {}
  submodule Buzz {
    // Equivalient to a file in current Swift.
    scope {
      // submodule declarations go here.
      // This is equivalent to the top level scope of a file that contains the `submodule Foo` declaration.
    }
    scope {}
    submodule Baz {}
  }
//}
```


## Appendix B: namespace style submodules

It is possible to design a system that allows a name boundary to be formed without also forming a scope boundary.  A natural consequence of this is that symbols may be placed into a namespace-style submodule in many (unlimited) scopes via extension (even extension outside the module is theoretically possible).  Allowing this runs contray to both of the two primary goals of this proposal (encapsulation and structure).

Allowing a submodule to be extended in multiple scopes precludes the possibility of submodule internal visibility.  A submodule internal access modifier could still be defined but it would not provide the guarantee it purports to.  The submodule can be opened by extension anywhere within the module.  If a lazy developer wants to access a submodule internal symbol from a distant subsytem all they need to do is add an extension and wrap the submodule internal symbol with a new symbol offering higher visibility*.  In such a system there is the same wide gap between file scope and module scope that exists today.

Allowing a submodule to be extended in multiple scopes precludes the ability to introduce real structure to a module.  We are able to introduce structure to the *names* but not the module itself.  The structure of a submodule in such a system may be widely dispersed throughout the module.  It is not part of a strictly hierarchical structure of scopes which each having a single designated location within the larger structure.

What you *do* get from name boundaries that do not also form a scope boundary is a soft form of symbol hiding (soft because *all* submodules are available for import or extension anywhere within the program).  This does provide some value, but not nearly as much value as is provided by a name boundary that is accompanied by a scope boundary.

Another downside to namespace-style submodules that are open to extension is that they are much less likely to facilitate improved build performance because they don't add any physical structure to the system.

Finally, lexical submodules have the potential to be confusing.  If submodules form a name boundary (even a soft one) an import statement is required to access the symbols declared inside a submodule.  Is code that surrounds a lexical submodule declaration able to see the symbols it declares without importing them?  Most developers will expect the symbols to be available.  It is probably necessary to make an exception to name boundary for the surrounding lexical context.  However, if an exception is made then this system relies heavily on a file to provide a bound to the implicit symbol import.

*It is worth observing that the ability to violate encapsulation via extension (or subclassing) is one of the primary reasons Swift does not offer type-based access modifiers such as `typeprivate` or `protected`.  The do not offer true encapsulation at all.  They are a statement of intent that cannot really be verified in the way that is desired.  They form a permeable rather than a hard boundary.

