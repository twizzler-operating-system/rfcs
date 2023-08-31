- Feature Name: twizzler_monitor
- Start Date: 2023-08-30
- RFC PR: [twizzler-rfcs/rfcs#0000](https://github.com/twizzler-operating-system/rfcs/pull/0000)
- Twizzler Issue: [twizzler-operating-system/twizzler#0000](https://github.com/twizzler-operating-system/twizzler/issues/0000)

# Summary
[summary]: #summary

This RFC describes the core functionality of what may be described as the "core Twizzler monitor" program. It proposes a formalized definition
of:
1. How programs (including libraries which may expose "nando"s, either secure or not) are **linked** and formed into objects.
2. How programs are **loaded**, including how they are isolated from each other.
3. What a program's **runtime** looks like, including the execution environment, threading, etc, and how the monitor supports programs.
4. How we achieve (optional) **isolation** between the loaded components of a running execution. 
5. How users might **interact** with this runtime at a high level.

A huge portion of this program is, essentially, a dynamic linker, or will have to be. However, it's a dynamic linker that is supported by
kernel-supported security contexts and gates, and a language (with runtime) that assists with both portability and abstraction.

# Motivation
[motivation]: #motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

* programmers expect a usable environment
* consoledate changes to low-level things like twizzler-abi
* make twizzler-abi abstract over a twizzler runtime, enables flexility
* unlocks a LOT of later tasks, while giving us all an execution environment that's easier to use and more flex to extend
* provides the core of support for what nandos, etc need to function
* demonstrates the usefulness of secure gate apis
* formalizes a core part of the system
* gives us a solid building point that starts with security

why a monitor
* thought excersize about levels of trust
* loader is a monitor that never gets called -- thus libc is a monitor
* we need it anyway?
* at least make the runtime flexible

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

* flex -- twizzler-abi provides runtime trait that std uses, and runtime monitors (or just loaders) can be swapped in and out. this is also how we swap in and out default naming systems. the default namer for twizzler-abi just looks at kernel init names, a runtime can instead interact with a naming service

* what the monitor does, at a high level: loads executables and libraries, facilitating isolation between components

* isolation levels

* how nandos are def and used (no isolation)

* how security gates are defined and used

* notes on allocation, stack, TLS, unwinding, global init/fini, global vars -- essentially the "irreducable rust runtime".

Explain the proposal as if it was already included in the system and you were teaching it to another Twizzler programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Twizzler programmers should *think* about the feature, and how it should impact the way they use Twizzler. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Twizzler programmers and new Twizzler programmers.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

* runtime operation overview

* containers

* executable objects, how they are linked

* how executable objects are loaded: incl how we intercept for perf, local alloc, etc

* how we deal with libraries: non-iso: nandos ;; iso: secure gates

* how secure gates work: how we handle each of "irreducable rust runtime" + accel (allocation). stack: via shadow stack, or new stack, or just on-stack, depending.

* how global, cli-callable nandos work (args are either String or ObjID, automatic lookup of name)

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
