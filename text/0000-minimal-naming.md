- Feature Name: minimal_naming
- Start Date: 2024-06-29
- RFC PR: [twizzler-rfcs/rfcs#0000](https://github.com/twizzler-operating-system/rfcs/pull/0000)
- Twizzler Issue: [twizzler-operating-system/twizzler#0000](https://github.com/twizzler-operating-system/twizzler/issues/0000)

# Summary
[summary]: #summary

Naming is a core feature in operating systems, and Twizzler is no exception.
One paragraph explanation of the feature.

# Motivation
[motivation]: #motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was already included in the system and you were teaching it to another Twizzler programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Twizzler programmers should *think* about the feature, and how it should impact the way they use Twizzler. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Twizzler programmers and new Twizzler programmers.



The Twizzler object abstraction uses a Foreign Object Table (FOT) to manage references between objects. Each FOT entry contains `(name-or-ID, flags, resolver)`, where `name-or-ID` refers
to either a 128-bit object ID or a pointer to a name and a length (a name is an arbitrary slice of bytes). The `resolver` field contains a pointer to a resolving function, and if non-null,
is used in place of the default pointer resolver. Pointer resolution is therefore _pluggable_, allowing arbitrary overriding behavior for data references.

The default resolver can handle both IDs and names natively. If the FOT entry contains an ID, the remaining bits of the invariant pointer (which has the form `(FOT-entry, offset)`) are
used as a direct byte-wise offset into the object. If the FOT entry contains a _name_, the default resolver invokes the runtime to resolve the name within the current address space. This is
done via _symbol lookup_ using the dynamic linker. The default resolver is thus capable only of resolving names that refer to global symbols loaded in from shared libraries or binaries. However,
since we can now point to _runtime functions_ via name from global (and namespaced) scope.

Let's look at an example. Say we have object A, and we want to have an FOT entry use a name with a custom resolver ("resolve_name" in crate "foo", which we'll say we loaded in via the dynamic linker). We can build that like so,
```
Object A's FOT
--------------
1 | "foo::resolve_name" | null
2 | "some name"         | InvPtr(1,0x0)
```

If we try to resolve a pointer, `InvPtr(2,0x1234)`, then the system will perform the following steps. Note that many of these are cachable, so they only need to happen once.

1. Lookup FOT entry 2, finding `resolver = InvPtr(1,0x0)`.
2. Lookup FOT entry 1, finding null.
3. Use the default resolver: `resolve("foo::resolve_name", 0x0)`, finding the address of the loaded shared object "foo" with offset `0x0`. 
4. Call the function at that address with arguments `("some name", 0x1234)`.

Another example is to enhance the behavior of references, for example, by reinterpreting the offset bits to be an _index_ in an array-based object.
```
Object B's FOT
--------------
1 | "twz_rt::resolve_idx" | null
2 | some-array-based-obj  | InvPtr(1,0x0)
```

If a pointer in object B (`InvPtr(2,21)`) is resolved, we first resolve in the same way as steps 1-3 above, but we end up calling `twz_rt::resolve_idx` instead. This function
finds the 21st entry in the referenced object, not the 21st byte, and returns a reference to it.

Note that this means that a resolution function can not only use the data in the name to influence where in an object we end up pointing to, but we can also interpret the offset bits of an invariant
pointer however we like. This kind of arbitrary behavior enables maximum flexibility while allowing us to easily map symbol reference and resolution to names in pointers, which ultimately allows for
referencing _resolvers themselves_ via function name.

 


# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?


Note that while this does, in essence, mean that we have persistent data pointing to runtime data, it isn't _really_. Firstly, it's a manual choice that is made, to use a name in an FOT entry. Secondly,
it refers to runtime data only via a fallible API (name resolution), and thus invalid names or missing functions can cause pointer resolution to fail gracefully.

This presents a minimal, memory-based API for references and names. The core runtime system, by itself, only has symbol lookup to go from for resolving names. This is especially true before any name resolving service
is loaded. The result is that the default pointer resolver, which requires minimal overhead to use as it can be optimized for the fast path, provides:

1. Direct memory-like resolution via ID + offset
2. Resolution of runtime resources via symbol lookup

Which is a minimal set of requirements to enable pluggable resolvers, and which fall natually out of this design.

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- Does this feature exist in other operating systems and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about
the lessons from other systems, provide readers of your RFC with a
fuller picture.  If there is no prior art, that is fine - your ideas
are interesting to us whether they are brand new or if it is an
adaptation from other operating systems.

Note that while precedent set by other operating systems is some
motivation, it does not on its own motivate an RFC.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal
would be and how it would affect the operating system and project as a
whole in a holistic way. Try to use this section as a tool to more
fully consider all possible interactions with the project and system
in your proposal.  Also consider how this all fits into the roadmap
for the project and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
