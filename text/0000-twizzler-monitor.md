- Feature Name: twizzler_monitor
- Start Date: 2023-08-30
- RFC PR: [twizzler-rfcs/rfcs#0000](https://github.com/twizzler-operating-system/rfcs/pull/0000)
- Twizzler Issue: [twizzler-operating-system/twizzler#0000](https://github.com/twizzler-operating-system/twizzler/issues/0000)

# Summary
[summary]: #summary

This RFC describes the core functionality of what may be described as the "core Twizzler monitor" program. It proposes a formalized definition
of:
0. How we can have a flexible runtime system that enables users to run "bare-metal" against twizzler-abi, against a more complete runtime, and even swap out the runtime system if they like.
1. How programs (including libraries which may expose "nando"s, either secure or not) are **linked** and formed into objects.
2. How programs are **loaded**, including how they are isolated from each other.
3. What a program's **runtime** looks like, including the execution environment, threading, etc, and how the monitor supports programs.
4. How we achieve (optional) **isolation** between the loaded components of a running execution. 
5. How users might **interact** with this runtime at a high level.

A huge portion of this program is, essentially, a dynamic linker, or will have to be. However, it's a dynamic linker that is supported by
kernel-supported security contexts and gates, and a language (with runtime) that assists with both portability and abstraction.

# Motivation
[motivation]: #motivation

As we are working towards an environment that enables programmers to write software for Twizzler, we need to set ourselves up
now for an extensible and well-defined runtime system and execution environment. In particular, this work will dramatically
improve the extensibility of the userspace parts of Twizzler and provide a core foundation upon which we can build higher-level
and secure software. In fact, a number of items in our respective roadmaps depend heavily on the functionality described by this RFC.

Additionally, this work will enable us to more easily use and demonstrate the usefulness of several core parts of the planned Twizzler programming
model, namely Nandos and Secure Gates. We also plan, as part of this work, to reach a semi-stable version of twizzler-abi, engineering this crate so
that the runtime can hook into core aspects of what Rust's standard library depend on and swap them out, allowing us more flexibility without having
to recompile programs for different runtimes.

Now, you may be asking, "why do we need a monitor". Well, lets consider the _minimum_ required secure environment that some sensitive software is running in.
In particular, notice that, in a traditional system, we must trust the kernel, the toolchain, the standard library (and all linked-to libraries), and the dynamic linker.
The issue of a trusted toolchain is out of scope of this RFC. We will focus instead on how we can ensure that the kernel, library, and dynamic linker can work together
to provide isolation. As a result, the dynamic linker for Twizzler will be Twizzler specific. This isn't particularly weird -- most dynamic linkers have a bunch of OS specific 
and arch specific code. Note that a traditional dynamic linker is already, kinda, a monitor, loading programs and libraries as required, etc.


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## The execution environment

The basic environment of a Twizzler program can be supported either by twizzler-abi directly or by a runtime which provides
some additional support on top of the basic twizzler-abi functionality. This may, for example, include access to a more useful
naming service, logging, debugging, networking, etc. Each piece of the system is defined as follows:

1. The twizzler-kernel provides the basic kernel services. It should only be accessed through twizzler-abi.
2. The twizzler-abi crate defines the interaction with the kernel, including syscall ABI, API, etc. It also defines the _Runtime_ trait.
3. The runtime trait provides an implementation of a twizzler runtime. It exposes the interface that Rust's standard library uses to interact with the runtime. A given implementation of runtime is selectable both at compile time _and_ at load time.
4. Twizzler ships with two runtimes: a default one provided by twizzler-abi that implements a bare minimum set of features to start up a program running against twizzler-abi. The second is the standard runtime that acts as a security monitor, dynamic linker, and program loader. It also facilitates secure gate calls between components.

```
+------------------------+   +-------------------+
|"Bare Metal" environment|   |Runtime Environment|
+---------+-+------------+   +-------+-+---------+
          | ^                        | ^
          | |                        | |
          v |                        v |
     +----+-+-------+         +------+-+---------+
     |              +-------->+                  |
     | twizzler-abi |         | twizzler-runtime |
     |              +<--------+                  |
     +------+-+-----+         +------------------+
            ^ |
            | |
            | v
  +---------+-+-----+
  |                 |
  | twizzler-kernel |
  |                 |
  +-----------------+
```

Here are two examples of how a runtime may offer functionality over that of the twizzler-abi runtime:

**Naming**: The runtime provides a default service for resolving names into object IDs. This requires a fair bit of runtime to work, as it needs a service running to
manage naming, persistence of naming, etc. However, we want Rust's std to be able to use this for paths too, so twizzler-abi needs to expose a hook for the more featureful
runtime to provide that functionality. What, then, does the "bare-metal" environment get? Well, those programmers could still implement their own naming system or hook into
one, but by default we'll have the twizzler-abi crate expose a read-only mapping of "initrd" names.

**stdout/stderr**: These, too, require a bit of runtime to make them as flexible as programmers generally expect them to be. The basic twizzler-abi runtime implements these by
writing to the kernel log. What else could it do? The more featureful runtime could collect output, stream it to different programs, log it, etc.

As a result, you can write a program for twizzler-abi and it is loadable into the more featureful runtime as well. The reverse may be true as well, though of course a program dependent
upon a specific runtime may not function correctly elsewhere.

## The Standard Runtime: loading programs

A core piece of the runtime is a dynamic linker. In particular, the runtime monitor loads and links programs and libraries both at load and run times. This means that 
the runtime needs to be able to load executable objects, map them, and (depending on the executable format) relocate them. Now, I'm not so crazy as to suggest we implement
our own executable format. ELF is a sufficiently weird machine that we'll probably get all the functionality we need out of it.

### So, what's a program?

The runtime supports both statically linked executables (or, dynamically linked traditional executables) and libraries (dynamic linked objects).
Executable programs have a defined entry point, libraries expose a set of symbols that are linked when the library is loaded. A program or library is contained within a single
object that contains an ELF file. This file is parsed and mapped into memory via Twizzler object APIs (notably, the copy-from primitive is used heavily along with direct mapping).
Note that not every ELF file will work. It does need to target Twizzler and link using the Twizzler linker script built into the Rust compiler's Twizzler support.

### The Loading Process: and example in a virtual address space

The monitor, when loading an executable program, does all the normal things a dynamic linker does, with the addition of a few things. First, it allows subcomponents of the
program and libraries to isolate from each other even within the same address space (this uses security contexts and secure gates). Let's imagine we have the following setup:

Program A links to libstd (S), libruntime (R), and a helper library, L. The resulting address space in Twizzler looks like:
```
+---------------------------+
|                           |
| A.text   A.data   A.state |
|                           |
|                           |
| R.text   R.data   R.state |
|                           |
|                           |
| L.text   L.data   L.state |
|                           |
|                           |
| S.text   S.data   S.state |
|                           |
|                           |
| stack    A.heap   thread  |
|                           |
|                           |
| M.text   M.data           |
|                           |
+---------------------------+
```
This is a bit of a simplification. Also note that I didn't really try to order these, as with Twizzler's invariant pointers + position-independent code, it doesn't matter. Well.
Okay, A.text and A.data, as the potentially statically-linked executable, may be forced into slots 0 and 1 respectively, but that's a small detail.

Each loaded executable object is split into two parts, text (executable code) and data (read-write data). We may add a third rodata object in the future. Each loaded object also has
a state object associated with it that the runtime uses to maintain state. We also have a heap (A.heap, we'll see why it's named as such later), a stack, a thread repr object (see the thread model documentation). We've loaded A and its dependencies (R, L, and S). We also have ourselves, the monitor / loader, as M, loaded.

Once everything is loaded and relocated (linked symbols, etc), we can start executing at A's entry point. During execution, A may decide to load more executable objects, or perform an
operation that causes the runtime to do so automatically. The result is the familiar programming environment we are all used to, in that we can construct statically-linked executables
or dynamically linked ones. We can load libraries at load or runtime, and use their exported symbols.

## The Standard Runtime: nandos

## The Standard Runtime: secure gates

One major functionality addition to the dynamic linker is compartmentalization. A given library may be called by a program, and that library
may be fully isolated from the caller. This is useful for building secure OS services, for example, a network library may manipulate configuration
data directly if it has write permission on that data. But your average program that _calls_ that library shouldn't be given write permission to that
data, as it could then trash it. Thus a key primitive that the runtime enables is for some insecure software to call a secure function in a library
that has some additional permissions not to be granted to the caller: a secure gated API.

Lets use the same example program and dependencies as from above with the following addition: The library L has additional permissions to update some
system state that the caller (A) doesn't have permission to do. Library L defines a function as follows:

```{rust}
#[twizzler_runtime::secure_gate]
fn update_some_state(args...) -> ... {...}
```

The caller can then just call `L::update_some_state(...)`. The runtime, during load, will ensure the following security properties:
1. The library L is protected from _arbitrary code execution_ by secure gates support in security contexts.
2. The "irreducible Rust Runtime" (discussed below) remains isolated within this library (we'll discuss what this means below).
3. The caller's stack cannot be corrupted (optional, adds overhead)
4. The callee's stack frames cannot be leaked to the caller (optional, adds overhead)
5. The caller's stack is not accessible at all by the callee (optional, limits API)

Once loaded, the virtual address space will contain:
```
+--------------------------------------------+
| thread(r--)                                |
| +----------------------------------------+ |
| |A.text(r-x)   A.data(rw-)   A.state(rw-)| |
| |                                        | |
| |A.heap(rw-)   stack(rw-)                | |
| |                                        | |
| |S.text(r-x)   S.data(rw-)   S.state(rw-)| |
| |                                        | |
| |R.text(r-x)   R.data(rw-)   R.state(rw-)| |
| +----------------------------------------+ |
|                                            |
|                                            |
| +----------------------------------------+ |
| |L.text(r-x)   L.data(rw-)   L.state(rw-)| |
| |                                        | |
| |L.heap(rw-)   L.stack(rw-)              | |
| |                                        | |
| |S.text(r-x)   S.data(rw-)   S.state(rw-)| |
| |                                        | |
| |R.text(r-x)   R.data(rw-)   R.state(rw-)| |
| +----------------------------------------+ |
|                                            |
+--------------------------------------------+
```
Note how there are multiple "containers" of objects, here. This has not actual bearing on virtual address space layout, they are compartments for
ensuring isolation. The specified permissions only apply to within a given compartment. The resulting, high-level view of the runtime monitor is then:
```
  +-----------------------------------------------+
  |                                               |
  |C_A: A's data, text, and non-isolated libraries|
  |                                               |
  +-----------------------------------------------+    +-----------+
                                       ^               |           |
                                       |               |  Monitor  |
                                       +<--------------+           |
                                       |               +-----------+
                                       v
  +-----------------------------------------------+
  |                                               |
  |C_B: B's data, text, and non-isolated libraries|
  |                                               |
  +-----------------------------------------------+
```
Wherein the monitor controls isolation between compartments, and within a compartment, it adds no overhead to control transfer between components. The resulting programming
model is one where a programmer can easily call functions exposed by system libraries that operate securely on protected data. These functions can go on to call other non-isolated
_or_ isolated functions, and the runtime will handle it.

### Isolation options

The minimum isolation required by a library can be set by the library at compile time, or by the caller of the library at load time. Once a library is
loaded, there is no way to change isolation levels without spinning up another compartmentalization of the same library.

There are, broadly, two major isolation directions to consider: does the caller trust the callee, does the callee trust the caller, or does neither party trust the other. The resulting table
shows the different options:

| Trust relationship          | Model 
|-----------------------------|------
| Both trust each other       | Simple dynamic library call
| Callee doesn't trust caller | Isolated callee (e.g. library to safely update system state)
| Caller doesn't trust callee | Isolated caller (e.g. program that operates on protected data calls an untrusted library)
| Neither trust the other     | Full isolation

* we can do different levels of isolation, each with their own tradeoffs (usually overhead)

## Notes on "thread state"

### The stack

stack is hard! but -- we can protect the stack so that on control transfer to another compartment, the stack is read-only, and the runtime creates a shadow stack on fault.

### The heap

each compartment has its own heap!

### TLS

same as heap!

### Unwinding

a secure gateway will need to catch_unwind

### global init/fini

on load time, this is easy

### Global variables

look, just dont.



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

* extension to secure gates as written in the paper: we'll need to protect thread-state-pointers (stack and base pointer, upcall pointer, and thread pointer) as well.

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
