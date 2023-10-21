# Feature name

* Proposal: [SE-NNNN](nnnn-protocol-based-attached-macros.md)
* Authors: [Joe Newton](https://github.com/SomeRandomiOSDev)
* Review Manager: TBD
* Status: **Awaiting implementation** or **Awaiting review**
* Vision: [Macros](https://github.com/apple/swift-evolution/blob/main/visions/macros.md)
* Upcoming Feature Flag: *N/A*
* Review: ([pitch](https://forums.swift.org/...))

When filling out this template, you should delete or replace all of
the text except for the section headers and the header fields above.
For example, you should delete everything from this paragraph down to
the Introduction section below.

As a proposal author, you should fill out all of the header fields
except `Review Manager`.  The review manager will set that field and
change several others as part of initiating the review.  Delete any
header fields marked *if applicable* that are not applicable to your
proposal.

When sharing a link to the proposal while it is still a PR, be sure
to share a live link to the proposal, not an exact commit, so that
readers will always see the latest version when you make changes.
On GitHub, you can find this link by browsing the PR branch: from the
PR page, click the "username wants to merge ... from username:my-branch-name"
link and find the proposal file in that branch.

`Status` should reflect the current implementation status while the
proposal is still a PR.  The proposal cannot be reviewed until an
implementation is available, but early readers should see the correct
status.

`Vision` should link to the [vision document](https://forums.swift.org/t/the-role-of-vision-documents-in-swift-evolution/62101)
for this proposal, if it is part of a vision.  Most proposals are not
part of a vision.  If a vision has been written but not yet accepted,
link to the discussion thread for the vision.

`Roadmap` should link to the discussion thread for the roadmap for
this proposal, if applicable.  When a complex feature is broken down
into several closely-related proposals to make evolution review easier
and more focused, it's helpful to make a forum post explaining what's
going on and detailing how the proposals are expected to be submitted
to review.  That post is called a "roadmap".  Most proposals don't need
roadmaps, but if this proposal was part of one, this field should link
to it.

`Bug` should be used when this proposal is fixing a bug with significant
discussion in the bug report.  It is not necessary to link bugs that do
not contain significant discussion or that merely duplicate discussion
linked somewhere else.  Do not link bugs from private bug trackers.

`Implementation` should link to the PR(s) implementing the feature.
If the proposal has not been implemented yet, or if it simply codifies
existing behavior, just say that.  If the implementation has already
been committed to the main branch (as an experimental feature), say
that and specify the experimental feature flag.  If the implementation
is spread across multiple PRs, just link to the most important ones.

`Upcoming Feature Flag` should be the feature name used to identify this
feature under [SE-0362](https://github.com/apple/swift-evolution/blob/main/proposals/0362-piecemeal-future-features.md#proposals-define-their-own-feature-identifier).
Not all proposals need an upcoming feature flag.  You should think about
whether one would be useful for your proposal as part of filling this
field out.

`Previous Proposal` should be used when there is a specific line of
succession between this proposal and another proposal.  For example,
this proposal might have been removed from a previous proposal so
that it can be reviewed separately, or this proposal might supersede
a previous proposal in some way that was felt to exceed the scope of
a "revision".  Include text briefly explaining the relationship,
such as "Supersedes SE-1234" or "Extracted from SE-01234".  If possible,
link to a post explaining the relationship, such as a review decision
that asked for part of the proposal to be split off.  Otherwise, you
can just link to the previous proposal.

`Previous Revision` should be added after a major substantive revision
of a proposal that has undergone review.  It links to the previously
reviewed revision.  It is not necessary to add or update this field
after minor editorial changes.

`Review` is a history of all discussion threads about this proposal,
in chronological order.  Use these standardized link names: `pitch`
`review` `revision` `acceptance` `rejection`.  If there are multiple
such threads, spell the ordinal out: `first pitch` `second review` etc.

## Introduction

Swift macros offers a variety of different macro types for performing code expansions for various use cases. One macro type that's discussed in the [Macros Vision document](https://github.com/apple/swift-evolution/blob/main/visions/macros.md) but not yet implemented is the `witness` macro type, which is an attached macro that synthesizes requirements for members its attached to in protocol declarations. 

While this attached macro type is certainly useful in its own right this document proposes a different, but related, macro type for declaring a protocol and its requirements where the invocation of the macro synthesizes the requirements of the protocol.

## Motivation

The kinds of macros that exist today make it simple to extend the Swift language in a highly customizable ways, which was previously only achievable through modification of the language/compiler itself. There's one feature, however, that's currently only available for select protocols that would be useful as its own macro type which we'll dub "Protocol Based Attached Macros."

These macros are associated directly with a given protocol and invoked when that protocol is attached to a particular type. This macro then generates the requirements of the protocol it's associated with, fulfilling the protocol's requirements. Two standout examples of this kind of macro (which is currently implemented as a compiler/language feature, not via a macro like this) are the `Encodable` and `Decodable` protocols:

```swift
protocol Encodable {
    func encode(to encoder: Encoder) throws
}

protocol Decodable {
    init(from decoder: Decoder) throws
}
```

When either of these protocols are attached to a given type, the compiler synthesizes the protocol's requirements in the type its attached to. Not only that, but the compiler performs validation of the type to ensure that each of its stored members conform to the protocol which is required for being able to automatically synthesize the conformance. If the requirements can't be automatically synthesized, the requirements can be manually implemented on the type. Not only that but the protocol's requirements can be manually implemented even if the requirements could be synthesized by the compiler.

This kind of functionality can already be somewhat emulated using the 'conformance' field on the `member` or `extension` attached macro type. For example, we could create an attached member macro that conforms the attached type to a given protocol and synthesizes the requirements of the protocol:

```swift
// Declaration
protocol FooBar {
    associatedtype Foo
    associatedtype Bar

    func foo() -> Foo
    func bar() -> Bar
}

@attached(member, conformances: FooBar, named: named(foo()), named(bar()))
macro FooBarMacro() = #externalMacro(...)

...

// Implementation
struct EncodableMacro: ExtensionMacro {
    static func expansion(
        of node: AttributeSyntax,
        attachedTo declaration: some DeclGroupSyntax,
        providingExtensionsOf type: some TypeSyntaxProtocol,
        conformingTo protocols: [TypeSyntax],
        in context: some MacroExpansionContext
    ) throws -> [ExtensionDeclSyntax] {
        return [
            ExtensionDeclSyntax(
                ...
                inheritanceClause:
                    InheritanceClauseSyntax(inheritedTypes: 
                        InheritedTypeListSyntax(protocols.map {
                            InheritedTypeSyntax(type: $0)
                        })
                    ),
                ...
            )
        ]
    }
}
```

Although this works fine, it does lack some of the functionality that makes these kinds of protocols what they are today. Specifically, if one directly conforms to a given protocol (e.g. `FooBar`) then the protocol's requirements aren't automatically synthesized like they would have been had the macro been used intstead:

```swift
struct FooBarImplA: FooBar {
    // error: Missing implementations for `foo()` and `bar()`
}

@FooBarMacro
struct FooBarImplB {

}

// Conformance to `FooBar` + requirements synthesized in an extension of `FooBarImplB`:
//
// extension FooBarImplB: FooBar {
//     func foo() -> Self.Foo {
//         ...
//     }
//     func bar() -> Self.Bar {
//         ...
//     }
// }
```

Furthermore, this workaround doesn't allow for the associated protocol to be composed into other protocols or types like the way that `Encodable` and `Decodable` are composed in the `Codable` typealias:

```swift
typealias HashableFooBar = (FooBar & Hashable)
// OR: `protocol HashableFooBar: FooBar, Hashable { }`

struct FooBarImpl: HashableFooBar {
    func hash(into hasher: inout Hasher) {
        ...
    }

    // error: Missing implementations for `foo()` and `bar()`
}
```

Because of these drawbacks we are proposing a new protocol attached macro type that mirrors the behavior of these kind of special protocols where the macro is invoked by conforming a type to the protocol.

## Proposed solution

The proposed solution is to create a new attached macro type `conformance` whose usage is restricted to protocol declarations:

```swift
@attached(conformance, ...)
protocol FooBar {
    ...
}
```

The implementation of the macro would then be invoked whenever the protocol is formally adopted on a conrete type, that is to say adopted by a class, struct, enum, or actor. When added to the inheritance clause of another protocol, the macro wouldn't be invoked. However, any conformance to the new protocol on a conrete would invoke the macro for that conrete type. For example:

```swift
@attached(conformance, ...)
protocol Foo {
    associatedtype Foo
    func foo() -> Foo
}

// `Foo`'s macro isn't invoked here
protocol FooBar: Foo {
    associatedtype Bar
    func bar() -> Bar
}

// `Foo`'s macro is invoked here and synthesizes the `foo()` requirement.
struct FooBarImpl: FooBar {
    func bar() -> Self.Bar {
        ...
    }
}
```

## Detailed design

The new `conformance` attached macro type would accept two different parameters in the attribute's usage:

1) `additionalMembers`

If provided, this field will accept a list of named parameters that specify any additional declarations that the macro will create. All of the declarations in the associated protocol are implicitly included and therefore it's not necessary to include them in this field. For example, if the `Encodable` protocol were declared as this kind of macro it would include an additional declaration for the `CodingKeys` enum that it creates:

```swift
@attached(conformance, additionalMembers: named(CodingKeys), macro: #externalMacro(...))
protocol Encodable {

    func encode(to encoder: Encoder) throws
}
```

2) `macro`

This field is required to be include in the attribute and is provided with the implementation specification of the macro:

```swift
@attached(conformance, macro: #externalMacro(...))
protocol FooBar {
    ...
}
```

Next, a new macro protocol type would be created in the `SwiftSyntax` library for specifying the implementation of the macro:

```swift
public protocol ProtocolConformanceMacro: AttachedMacro {

    static func expansion(
        of protocol: ProtocolDeclSyntax,
        attachedTo declaration: some DeclGroupSyntax,
        in context: some MacroExpansionContext
    ) throws -> [DeclSyntax]
}
```

The parameters of the required expansion function are defined as follows:

- `protocol`:

In lieu of providing an `AttributeSyntax` like is done in all other attached macros, this function accepts the declaration of the protocol that this macro is being invoked.

- `declaration`:

The `declaration` field is provided with the concrete declaration type that the protocol is attached to. The AST this declaration would be equivalent to the declaration that would be provided to the invocation of a `member` or `extension` macro. That is to say that the full AST of the conforming type would be provided, not just the type of the declaration or a "skeleton" of the declaration.

- `context`:

The standard macro expansion context included in all other macro expansion functions.

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

While this attached macro type would certainly be useful for more complex protocols with multiple required members where we wanted to synthesize each of the members, creating a macro for each of the protocol's members would become quite cumbersome and polute the global namespace with a number of different single-use macros. It we were to instead create a single attached macro that we would attach to each of the different declarations we want to synthesize, the implementation of the macro could become exceeding complex to account for all of the different members that it's supposed to generate, assuming that we could even structure a macro in this way.

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
