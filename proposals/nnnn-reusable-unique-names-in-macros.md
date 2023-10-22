# "Re-usable" unique names in Macros.

* Proposal: [SE-NNNN](nnnn-reusable-unique-names-in-macros.md)
* Authors: [Joe Newton](https://github.com/SomeRandomiOSDev)
* Review Manager: TBD
* Status: **Awaiting implementation** or **Awaiting review**
* Vision: [Macros](https://github.com/apple/swift-evolution/blob/main/visions/macros.md)
* Implementation: TBD
* Review: ([pitch](https://forums.swift.org/...))

## Introduction

This document outlines a proposal for adding support for creating unique names in macro expansions that can be re-used across invocations of different roles for the same macro invocation.

## Motivation

Today the [MacroExpansionContext](https://github.com/apple/swift-syntax/blob/f8be751eaefe0d7c72c021f12024a8328a5f3568/Sources/SwiftSyntaxMacros/MacroExpansionContext.swift#L19C23-L19C23) protocol provides an API for generating unique names that are unique to a given macro expansion: [makeUniqueName(_:)](https://github.com/apple/swift-syntax/blob/f8be751eaefe0d7c72c021f12024a8328a5f3568/Sources/SwiftSyntaxMacros/MacroExpansionContext.swift#L28C13-L28C13). 

This function, as the name suggests, creates names (identifiers) that are unique to the macro's expansion for a given macro role. The unique names generated follow a specific pattern which is outlined for the `macro-expansion-operator` entity in the [Mangling](https://github.com/apple/swift/blob/main/docs/ABI/Mangling.rst#entities) document:

```
macro-expansion-operator ::= decl-name identifier 'fMa' // attached accessor macro
macro-expansion-operator ::= decl-name identifier 'fMr' // attached member-attribute macro
macro-expansion-operator ::= identifier 'fMf' // freestanding macro
macro-expansion-operator ::= decl-name identifier 'fMm' // attached member macro
macro-expansion-operator ::= decl-name identifier 'fMp' // attached peer macro
macro-expansion-operator ::= decl-name identifier 'fMc' // attached conformance macro
```

For example, for the given macro some unique names that it generates might look at follows:

```swift
// Declaration
@attached(member, names: arbitrary)
macro FooBarMacro(...) = #externalMacro(...)

// In module "MyModule":
@FooBarMacro
struct FooBar {
    ...
}

// Implementation of `FooBarMacro`:
struct FooBarMacro: MemberMacro {
    static func expansion(
        of node: AttributeSyntax,
        providingMembersOf declaration: some DeclGroupSyntax,
        conformingTo protocols: [TypeSyntax],
        in context: some MacroExpansionContext
    ) throws -> [DeclSyntax] {
        let uniqueName1 = context.makeUniqueName("foobar")
        // uniqueName1 == "$s8MyModule6FooBarfMm_6foobarfMu_"

        let uniqueName2 = context.makeUniqueName("foobar")
        // uniqueName2 == "$s8MyModule6FooBarfMm_6foobarfMu0_"

        let uniqueName3 = context.makeUniqueName("baz")
        // uniqueName3 == "$s8MyModule6FooBarfMm_3bazfMu_"
    }
}
```

As outlined above, these mangled unique names are specific to the role of the macro being expanded. This is where we can run in issues for specific use cases. For example, let's say that we wanted to create a macro for synthesizing accessors on a variable for getting and setting its value as an associated object on the type. An example expansion of this macro might look like follows:

```swift
// Macro declaration
@attached(peer, arbitrary)
@attached(accessor)
macro AssociatedObject() = #externalMacro(...)

// Macro usage (located in module "MyModule")
class MyClass {
    @AssociatedObject
    var foobar: AnyObject?
}

// expands to:

class MyClass {
    private static let <uniqueName>: UnsafeRawPointer = ...

    var foobar: AnyObject? {
        get { objc_getAssociatedObject(self, Self.<uniqueName>) }
        set { objc_getAssociatedObject(self, Self.<uniqueName>, newValue, .OBJC_ASSOCIATION_POLICY) }
    }
}
```

The implementation of this macro will conform to the `PeerMacro` and the `AccessMacro` protocols to fulfil the two macro roles that are declared on the macro. The implementation of the `PeerMacro` is responsible for creating the declaration of the static variable for the associated object key. In doing this it needs to create a unique identifier. The identifier might look something like the following:

```swift
extension AssociatedObjectMacro: PeerMacro {
    static func expansion(
        of node: AttributeSyntax,
        providingPeersOf declaration: some DeclSyntaxProtocol,
        in context: some MacroExpansionContext
    ) throws -> [DeclSyntax] {
        let uniqueVariableName = context.makeUniqueName("associatedObjectKey")
        // uniqueVariableName == "$s8MyModule7MyClass6foobarfMp_19associatedObjectKeyfMu_"
    }
}
```

Next in the implementation of the `AccessorMacro` we will create the accessors for getting and setting the variable's value. These accessors will need to reference the unique name of the variable we created in the implementation of the `PeerMacro`:

```swift
extension AssociatedObjectMacro: AccessorMacro {
    static func expansion(
        of node: AttributeSyntax,
        providingAccessorsOf declaration: some DeclSyntaxProtocol,
        in context: some MacroExpansionContext
    ) throws -> [AccessorDeclSyntax] {
        let uniqueVariableName = context.makeUniqueName("associatedObjectKey")
        // uniqueVariableName == "$s8MyModule7MyClass6foobarfMa_19associatedObjectKeyfMu_"
    }
}
```

Carefully inspecting the variable names, however, we see that they are actually different given that the prefixes used for each of the roles differ by a single letter:

```swift
// AccessorMacro
$s8MyModule7MyClass6foobarfMa_19associatedObjectKeyfMu_
                          ^~~~

// Peer macro
$s8MyModule7MyClass6foobarfMp_19associatedObjectKeyfMu_
                          ^~~~
```

Becuase of this minute discrepancy the generated accessors will attempt to access a variable that doesn't exist in the expansion:

```swift
class MyClass {
    private static let $s8MyModule7MyClass6foobarfMp_19associatedObjectKeyfMu_: UnsafeRawPointer = ...

    var foobar: AnyObject? {
        get { objc_getAssociatedObject(self, Self.$s8MyModule7MyClass6foobarfMa_19associatedObjectKeyfMu_) }
                                                  ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
                                                  // error: Type 'MyClass' has no member named '$s8MyModule7MyClass6foobarfMa_19associatedObjectKeyfMu_'
        set { objc_getAssociatedObject(self, Self.$s8MyModule7MyClass6foobarfMa_19associatedObjectKeyfMu_, newValue, .OBJC_ASSOCIATION_POLICY) }
                                                  ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
                                                  // error: Type 'MyClass' has no member named '$s8MyModule7MyClass6foobarfMa_19associatedObjectKeyfMu_'
}
```

## Proposed solution

To account for cases like the one described above, we need to extend the `makeUniqueName(_:)` function, or create a new one, that allows for the creation of "re-usable" unique identifiers that will guaranteed to be equivalent across different roles of the same macro invocation. The proposed solution is to add an optional flag to the `makeUniqueName(_:)` function that, if provided and set to `true`, will create a unique name that follows the same pattern as above but will be the same when invoked from other macro role implementations:

```swift
protocol MacroExpansionContext {

    func makeUniqueName(_ name: String, reusable: Bool) -> TokenSyntax
                                        ^~~~~~~~~~~~~~ // new parameter

    ...
}
```

Describe your solution to the problem. Provide examples and describe
how they work. Show how your solution is better than current
workarounds: is it cleaner, safer, or more efficient?

This section doesn't have to be comprehensive.  Focus on the most
important parts of the proposal and make arguments about why the
proposal is better than the status quo.

## Detailed design

Describe the design of the solution in detail. If it involves new
syntax in the language, show the additions and changes to the Swift
grammar. If it's a new API, show the full API and its documentation
comments detailing what it does. The detail in this section should be
sufficient for someone who is *not* one of the authors to be able to
reasonably implement the feature.

## Source compatibility

Describe the impact of this proposal on source compatibility.  As a
general rule, all else being equal, Swift code that worked in previous
releases of the tools should work in new releases.  That means both that
it should continue to build and that it should continue to behave
dynamically the same as it did before.  Changes that cannot satisfy
this must be opt-in, generally by requiring a new language mode.

This is not an absolute guarantee, and the Language Workgroup will
consider intentional compatibility breaks if their negative impact
can be shown to be small and the current behavior is causing
substantial problems in practice.

For proposals that affect parsing, consider whether existing valid
code might parse differently under the proposal.  Does the proposal
reserve new keywords that can no longer be used as identifiers?

For proposals that affect type checking, consider whether existing valid
code might type-check differently under the proposal.  Does it add new
conversions that might make more overload candidates viable?  Does it
change how names are looked up in existing code?  Does it make
type-checking more expensive in ways that might run into implementation
limits more often?

For proposals that affect the standard library, consider the impact on
existing clients.  If clients provide a similar API, will type-checking
find the right one?  If the feature overloads an existing API, is it
problematic that existing users of that API might start resolving to
the new API?

## ABI compatibility

Describe the impact on ABI compatibility.  As a general rule, the ABI
of existing code must not change between tools releases or language
modes.  This rule does not apply as often as source compatibility, but
it is much stricter, and the Language Workgroup generally cannot allow
exceptions.

The ABI encompasses all aspects of how code is generated for the
language, how that code interacts with other code that has been
compiled separately, and how that code interacts with the Swift
runtime library.  Most ABI changes center around interactions with
specific declarations.  Proposals that do not affect how code is
generated to interact with an external declaration usually do not
have ABI impact.

For proposals that affect general code generation rules, consider
the impact on code that's already been compiled.  Does the proposal
affect declarations that haven't explicitly adopted it, and if so,
does it change ABI details such as symbol names or conventions
around their use?  Will existing code change its dynamic behavior
when running against a new version of the language runtime or
standard library?  Conversely, will code compiled in the new way
continue to run on old versions of the language runtime or standard
library?

For proposals that affect the standard library, consider the impact
on any existing declarations.  As above, does the proposal change symbol
names, conventions, or dynamic behavior?  Will newly-compiled code work
on old library versions, and will new library versions work with
previously-compiled code?

This section will often end up very short.  A proposal that just
adds a new standard library feature, for example, will usually
say either "This proposal is purely an extension of the ABI of the
standard library and does not change any existing features" or
"This proposal is purely an extension of the standard library which
can be implemented without any ABI support" (whichever applies).
Nonetheless, it is important to demonstrate that you've considered
the ABI implications.

If the design of the feature was significantly constrained by
the need to maintain ABI compatibility, this section is a reasonable
place to discuss that.

## Implications on adoption

The compatibility sections above are focused on the direct impact
of the proposal on existing code.  In this section, describe issues
that intentional adopters of the proposal should be aware of.

For proposals that add features to the language or standard library,
consider whether the features require ABI support.  Will adopters need
a new version of the library or language runtime?  Be conservative: if
you're hoping to support back-deployment, but you can't guarantee it
at the time of review, just say that the feature requires a new
version.

Consider also the impact on library adopters of those features.  Can
adopting this feature in a library break source or ABI compatibility
for users of the library?  If a library adopts the feature, can it
be *un*-adopted later without breaking source or ABI compatibility?
Will package authors be able to selectively adopt this feature depending
on the tools version available, or will it require bumping the minimum
tools version required by the package?

If there are no concerns to raise in this section, leave it in with
text like "This feature can be freely adopted and un-adopted in source
code with no deployment constraints and without affecting source or ABI
compatibility."

## Future directions

Describe any interesting proposals that could build on this proposal
in the future.  This is especially important when these future
directions inform the design of the proposal, for example by making
sure an attribute encodes enough information to be used for other
purposes.

The rest of the proposal should generally not talk about future
directions except by referring to this section.  It is important
not to confuse reviewers about what is covered by this specific
proposal.  If there's a larger vision that needs to be explained
in order to understand this proposal, consider starting a discussion
thread on the forums to capture your broader thoughts.

Avoid making affirmative statements in this section, such as "we
will" or even "we should".  Describe the proposals neutrally as
possibilities to be considered in the future.

Consider whether any of these future directions should really just
be part of the current proposal.  It's important to make focused,
self-contained proposals that can be incrementally implemented and
reviewed, but it's also good when proposals feel "complete" rather
than leaving significant gaps in their design.  For example, when
[SE-0193](https://github.com/apple/swift-evolution/blob/main/proposals/0193-cross-module-inlining-and-specialization.md)
introduced the `@inlinable` attribute, it also included the
`@usableFromInline` attribute so that declarations used in inlinable
functions didn't have to be `public`.  This was a relatively small
addition to the proposal which avoided creating a serious usability
problem for many adopters of `@inlinable`.

## Alternatives considered

Describe alternative approaches to addressing the same problem.
This is an important part of most proposal documents.  Reviewers
are often familiar with other approaches prior to review and may
have reasons to prefer them.  This section is your first opportunity
to try to convince them that your approach is the right one, and
even if you don't fully succeed, you can help set the terms of the
conversation and make the review a much more productive exchange
of ideas.

You should be fair about other proposals, but you do not have to
be neutral; after all, you are specifically proposing something
else.  Describe any advantages these alternatives might have, but
also be sure to explain the disadvantages that led you to prefer
the approach in this proposal.

You should update this section during the pitch phase to discuss
any particularly interesting alternatives raised by the community.
You do not need to list every idea raised during the pitch, just
the ones you think raise points that are worth discussing.  Of course,
if you decide the alternative is more compelling than what's in
the current proposal, you should change the main proposal; be sure
to then discuss your previous proposal in this section and explain
why the new idea is better.

## Acknowledgments

If significant changes or improvements suggested by members of the 
community were incorporated into the proposal as it developed, take a
moment here to thank them for their contributions. Swift evolution is a 
collaborative process, and everyone's input should receive recognition!

Generally, you should not acknowledge anyone who is listed as a
co-author or as the review manager.
