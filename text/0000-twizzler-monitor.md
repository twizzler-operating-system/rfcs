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

The runtime is a program that acts as a security monitor and dynamic linker. It loads programs and libraries into memory and executes them. Any program
or library that is loaded by the runtime is then under the control of that runtime. The runtime organizes programs and libraries that are loaded into
_compartments_, each one providing a configurable level of isolation from the others for the programs and libraries residing within.

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
+-----------------+
|                 |
| A.text   A.data |
|                 |
| R.text   R.data |
|                 |
| L.text   L.data |
|                 |
| S.text   S.data |
|                 |
| stack    A.heap |
|                 |
| M.text   M.data |
|                 |
| thread          |
+-----------------+
```
This is a bit of a simplification. Also note that I didn't really try to order these, as with Twizzler's invariant pointers + position-independent code, it doesn't matter. Well.
Okay, A.text and A.data, as the potentially statically-linked executable, may be forced into slots 0 and 1 respectively, but that's a small detail.

Each loaded executable object is split into two parts, text (executable code) and data (read-write data). We may add a third rodata object in the future. We also have a heap (A.heap, we'll see why it's named as such later), a stack, a thread repr object (see the thread model documentation). We've loaded A and its dependencies (R, L, and S). We also have ourselves, the monitor / loader, as M, loaded.

Once everything is loaded and relocated (linked symbols, etc), we can start executing at A's entry point. During execution, A may decide to load more executable objects, or perform an
operation that causes the runtime to do so automatically. The result is the familiar programming environment we are all used to, in that we can construct statically-linked executables
or dynamically linked ones. We can load libraries at load or runtime, and use their exported symbols.

## The Standard Runtime: nandos

For now, until Achilles and I flesh this out, let's just imagine nandos as a function and some associated data:

```{rust}
#[nando]
pub fn foo(...) -> ... {...}
```

This will generate a FooNando struct that the runtime can hook into to understand some details about this nando. However, from a programmer's perspective, they can just do something like `nando!(lib::foo)`, or just `lib::foo(...)`, etc, to invoke it. If the library has no security restrictions, this can be super cheap (either statically linked into the program, or a simple dynamic library call). If it has restrictions, however...

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
| +--------------------------+               |
| |A.text(r-x)   A.data(rw-) |               |
| |                          |               |
| |A.heap(rw-)   stack(rw-)  |               |
| |                          |               |
| |S.text(r-x)   S.data(rw-) |               |
| |                          |               |
| |R.text(r-x)   R.data(rw-) |               |
| +--------------------------+               |
|                                            |
|                                            |
| +---------------------------+              |
| |L.text(r-x)   L.data(rw-)  |              |
| |                           |              |
| |L.heap(rw-)   L.stack(rw-) |              |
| |                           |              |
| |S.text(r-x)   S.data(rw-)  |              |
| |                           |              |
| |R.text(r-x)   R.data(rw-)  |              |
| +---------------------------+              |
|                                            |
+--------------------------------------------+
```
Note how there are multiple "compartments" of objects, here. This has not actual bearing on virtual address space layout, they are compartments for
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

The anatomy of a compartment is as follows:
```
           +---------+
           | program |
           +--------++
                    ^
                    |
                    |
                    v
+------------+     ++-------+
| libruntime +<--->+ libstd |
+----+-------+     +--------+
     |
     v
+----+--+     +------+     +-------+
| state +---->+ heap |     | stack |
+-------+     +------+     +-------+
```

A program (which may be a library) links against libstd, which links against twizzler-abi (not shown) and libruntime. The runtime library then uses the state object (one per compartment) to locate the heap
and handle allocation requests from the standard library.

### Isolation options

The minimum isolation required by a library can be set by the library at compile time, or by the caller of the library at load time. Once a library is
loaded, there is no way to change isolation levels without spinning up another compartmentalization of the same library.

There are, broadly, two major isolation directions to consider: does the caller trust the callee, does the callee trust the caller, or does neither party trust the other. The resulting table
shows the different options:

| Trust relationship          | Model 
|-----------------------------|------
| Both trust each other       | No isolation -- simple dynamic library call
| Callee doesn't trust caller | Isolated callee (e.g. library to safely update system state)
| Caller doesn't trust callee | Isolated caller (e.g. program that operates on protected data calls an untrusted library)
| Neither trust the other     | Full isolation

Whether or not the callee trusts the caller is set by a flag in the secure gate itself, so the caller can never bypass the isolation by setting a different policy. Similarly, the
caller can specify its trust level at load or runtime.

Selection of higher isolation levels may come with a tradeoff. In particular, performance and restrictions on what APIs are possible. They are as follows:

1. No isolation: no overhead above a standard dynamic library call
2. Any amount of isolation requires security context switches and shadow stack construction.
3. If the caller doesn't trust the callee to even see its stack, then the callee must run on a brand new stack, so APIs may be restricted to not use the stack for argument passing in this case, although we may be able to get around this with some clever proc macros and trampolining.

## Notes on "thread state" and secure gates

When switching between compartments, we must be careful to avoid 1) improper transfer of control flow to a compartment, 2) isolation of sensitive information within a compartment, and 3) prevention of the isolated compartment getting "tricked" into writing somewhere it didn't intend. The details of _how_ we'll deal with these will be described by the reference level explanation below, but here we'll quickly discuss some basic concepts. 

### The stack

One key aspect of thread runtime state that we don't have direct control over normally is the stack. Instructions generated by the compiler interact with the stack frequently and in automatic ways relative to our level of abstraction in programming Rust. Thus we, as programmers, have little control over when and how the stack is accessed, and cannot ensure that we get to "check" that the stack pointer is "okay" before its used -- after all, a checking function will still use the stack!

Why do we care? Well, imagine an attack where an untrusted program calls a sensitive, isolated library to modify data in object O. Normally, the untrusted program cannot access object O, but imagine if we pointed our stack pointer into O and then called the secure gate! The isolated library will then corrupt O with stack frame data before it even gets a chance to do anything else.

Thus the runtime will ensure that the isolated library cannot accidentally corrupt some object using its stack pointer (details later).

Another concern is privacy -- any stack frames pushed to the stack by the isolated library could be visible to the untrusted program if they shared stacks. For this reason, the runtime
sets up a shadow stack so that pushed stack frames by the isolated library stay within that library, and any stack arguments are still accessible. Note, however, that this now restricts using the stack for return values. We'll need to figure this one out.

### The heap

The heap is another big issue. See, Rust follows the common programming model of a global heap, one per process. If we were to continue with this model, we'd need to put the APIs for accessing the global heap behind secure gates into the runtime, and even then we'd need _hardware_ capabilities to prevent isolated compartments from corrupting heap data. Instead,
we'll change the model to one heap per compartment, that way allocation can happen without huge overhead.

One thing this means is that heap data may, sometimes, not be sharable across compartments. We propose to default to a private-heap-per-compartment, so normal allocations are contained,
however we can use Rust's allocator API to allocate from heaps that can be shared across compartments.

### TLS

Thread-local storage is another major concern, as it involves the use of an architectural thread-pointer that we need to ensure has similar restrictions as the stack.

### Unwinding

Unwinding across a foreign function interface is undefined behavior in Rust. While we're not _exactly_ crossing an FFI boundary, we kinda are, since the heap and stack pointer change
dramatically between compartments. On top of that, as we'll see later, the way we return from a compartment requires a bit of extra work. Thus we must always catch an unwind before going
back across the compartment boundary. The runtime will allow the unwind to be caught, and resumed, across the boundary.

### Global variables

Look, if you make a global variable, and then try to share it across a compartment boundary, it'll be restricted (either read-only or inaccessible), so I guess just plan for that.
Or just don't use global, shared variables.

## The Reference REPL

As one example of something you could build atop this, let's consider an interactivity model for Twizzler. Instead of a "shell", we'll think of the interaction point as a REPL, broadly
defined, to be defined concretely in a different RFC. We'll consider some example, shell-like syntax here as a placeholder to avoid bikeshedding. Consider that, in a system where libraries explicitly expose calling points (Nandos, Secure Gates), we could expose these _typed_ interaction points to the command line interface itself. For example, imagine a library exposes an interface for updating the password as follows:

```{rust}
#[secure_gate]
pub fn update_password(user: &User, password: String) -> Result<(), UpdatePasswordError>;
pub fn lookup_user(name: &str) -> Result<User, LookupUserError>;
```

Not saying this is a _good_ interface, just roll with it. Let's imagine that the `User` type implements `TryFrom<String>`, and the `UpdatePasswordError` and `LookupUserError` types are enums with variants listing possible failures. Further, these error types implement the Error trait. Now, let's say these functions are in a library that gets compiled to an object named `libpasswd`. We can then expose this library to the REPL. The REPL can enumerate the interface for the library. If the source is provided (or, maybe if we look at rust metadata??? or debug data??? or we generate type info in the nando struct???), the REPL _knows_ the interface _and_ all the types for the functions, so it can extrapolate an interface for the command line automatically:

Long form (the `X ?= Y` syntax is sugar for `X = Result::unwrap (Y)`):
```
twz> user ?= passwd::lookup_user bob
twz> passwd::update_passwd user changeme
Ok(())
twz>
```

Wrong username:
```
twz> user ?= passwd::lookup_user bobby
Error: No such user
```

If we hadn't implemented `TryFrom<String>` for `User`
```
twz> passwd::update_passwd bob changeme
Type Error: Expected 'User' got 'String'
```

But, since we did, we get a shortcut:
```
twz> passwd::update_passwd bob changeme
Ok(())
```

Or, with the wrong username:
```
twz> passwd::update_passwd bobby changeme
Error: No such user
```

Or, if you don't have permission:
```
twz> passwd::update_passwd bob changeme
Security Error: Failed to call update_password in passwd: Permission denied
```

The basic REPL here has special handling for String, Into/From/TryInto/TryFrom String, impl Error, Result, Option, etc, but otherwise builds this command line interface, including arguments and error reporting, automatically from the types.

Another example would be some library, `foo`, that wants to update some object by ID. So it exposes some function `bar(id: ObjID) -> ...`. Now, typing an object ID is annoying, but doable
if necessary, so we do allow that. But the REPL can see this type and automatically try to resolve that positional argument with a (default, but configurable) name resolution mechanism. So the user can still type a name, even if the actual programming interface takes an object by ID. And this can be extended to object _handles_, too. Those can even be typed:

```
#[nando]
pub fn baz<T: SomeKindaObject>(obj: Object<T>) -> ...;
```

If "name" resolves to an object that implements `SomeKindaObject`:
```
twz> foo::baz name
...
```

If "name" resolves to an object that does NOT implement `SomeKindaObject`:
```
twz> foo::baz name
Type Error: Object 'name' does not implement SomeKindaObject.
```

If "name" does not resolve:
```
twz> foo::baz name
Name Error: Object 'name' failed to resolve: ...
```

This means that system software _is_ the libraries written to operate or interact with the system. The command line interface is just a translation from command-line interface interaction to library calls, for which the Type info is sufficient to automatically generate.

And of course a more powerful REPL can just expose library calls that interact with the system and the data model directly in its programming language.

Note that you can recover the semantics of a standard unix-like program via `fn cli_main(args: &[&str]) -> i32`, and in fact, this would be an effective wrapper around such programs as a way to invoke them (via just loading it, and passing args via ELF aux data, or whatever).

Also note that I'm just using this above syntax as an example -- one powerful feature of this is not just making scripting the OS even easier than shell scripts, as you get typed library calls instead of invocation of an executable, but doing so without coupling deeply to the _language_.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Phew, okay. Let's start in on how we can make this happen.

The runtime is a combination security monitor and dynamic linker, managing the loading and linking of programs and libraries within and across
isolation domains called _compartments_. It is, explicitly, part of the trusted computing base.

## Security Contexts

Twizzler offers an abstraction of a security context with the following properties:
1. A context contains capabilities that define the access rights of a thread that has this context as active.
2. A thread can be _attached_ to multiple contexts, though only one may be active at one time.
3. A thread switches contexts automatically if a security fault occurs, and the thread is attached to a context in which the instruction would not cause a security fault.
4. Transfer of control between contexts is limited by an object's _secure gates_, a set of allowed entry points into this object. If a security context switch occurs, and the instruction pointer is not pointing to a valid gate, that context is not considered a valid context to switch to.

The runtime uses this feature via a 1-1 mapping of compartment to security context. All programs and libraries within a given compartment all run within the same security context (if possible), or others with control flow transfer managed by the runtime if necessary. Note that the runtime itself is contained within a compartment and isolated from all other compartments, allowing only approved operations to be invoked (via security gates) should a loaded program wish to interact with the runtime monitor directly.

## How to isolate a library

Let's go back to that running example: untrusted program A wishes to call isolated library L's function foo, which is a security gate. First, library L (and its associated runtime state) are contained within compartment CL, while A is contained within CA. Upon call to CL, via the call instruction, the processor pushes the return address to the stack and then jumps. The
instruction pointer is now in L, pointing at the start of foo. But we are still within context CA! This triggers a page fault, since, because it is isolated, L is not executable in CA.
The kernel finds CL, where the execution is valid, and then continues with the first instruction. Let's say it's `push rbp`, which it probably is. This triggers a security fault. Why?

See, we must protect against the callee (foo) corrupting some data that A didn't have write access to but L does. The caller could have pointed the stack pointer at some sensitive data and then called foo. To protect against this, the stack that A is using is not writable in CL. The security fault is handled by the runtime monitor, as it's registered itself as the fault handler with the kernel during setup. The runtime monitor then constructs a shadow stack for L, using the object copy-from primitive. Execution is then allowed to proceed normally.

Upon return, we have to do another thing -- we have to check to see if the return address is safe. The caller _could_ have pushed a location within L and then _jumped_ to foo, instead of calling it. This would re-enable arbitrary code execution in L! So the runtime, when constructing the shadow stack, checks the return address. Finally, what if foo just instantly and blindly executes ret? While unlikely, we still have to deal with this. By default, the stack is made not readable from CL, however the runtime still constructs the shadow stack. This option doesn't prevent L from reading A's stack frames, it just prevents L from doing anything with A's stack data before the runtime can interpose.

Finally, we can optionally refuse to create a shadow stack and instead require that the callee has a new stack. This results in the construction of a new stack with a fake frame that returns to the call site, but contains no other data from A's frames.

### So, how do we enforce that the stack isn't writable on context switch?

Yeah, we do need to do this! See, the "prevent corruption" motivation above does still require the runtime to interpose always, which means that the stack pointer _cannot_ be writable (or, even readable sometimes!) when a gate is used. Thus we propose to extend the Twizzler gate model to include the ability to check the validity of _architectural pointers_:

* Stack and base pointer: at least -w- in source and at most r-- in target
* Thread pointer: at most r-- in both contexts. This covers TLS -- the thread pointer points to a region of memory that is used as the TLS block locator for dynamic memory objects. That index should not be writable by either context, as it is under the control of the runtime. Further, the thread pointer should not be _changeable_ in the source context (denoted by the capability to write the thread repr object).
* Upcall pointer (not really architectural, but): at most r-- in both contexts. The runtime needs to be the handler for upcalls, so we ensure that it cannot be executable in the target, nor do we want the source compartment to handle (e.g.) security violations for the target compartment. Same as thread pointer: not changeable in the source context.

For flexibility, these permissions should not be hardcoded, but instead will allow the gate's creator to specify both required and disallowed masks. However, the above may be the default.

### What about the heap?

Each compartment has a heap, which means each compartment has an independent allocator and libstd. We can achieve this by linking to a compartment-local libruntime that is configured with enough information to manage its own heap independent of the other compartments. This is a different linking algorithm than is standard for dynamic libraries. We are first explicitly linking against a library that has "first dibs" on symbols, and falling back to global symbol resolution only after that (ok -- I suppose this is kinda like LD_PRELOAD).

This trick, of allowing heap allocation without the runtime, is necessary for correctness (isolation of heap data) but _also_ for performance, since a context switch on every allocation would be a hard pill to swallow. Fortunately, we can make heap allocation just as cheap as it is now with this per-compartment heap trick, at the cost of some additional complexity to share heap data across compartments. Heaps are private by default, but we can always create additional heaps with different permissions and then use Rust's allocator API to allocate data from those heaps instead of the default.

## How are executable objects linked and loaded?

We use the linker provided by LLVM to link executables and libraries, however we provide a custom linker script that ensures that the memory layout of the program fits with the runtime.
For example, a typical program might look like this:

```
| Section | vaddr  | len   | offset | perms |
|---------|--------|-------|-------:|:-----:|
| .text   | 0x1000 | 0x800 |      0 |  r-x  |
| .rodata | 0x1800 | 0x100 |  0x800 |  r--  |
| .data   | 0x2000 | 0x100 | 0x1000 |  rw-  |
| .bss    | 0x2100 | 0x130 |    N/A |  rw-  |
```

As you can see, the program's sections are loaded into specific memory addresses, with data taken from the offset of the file (this is a simplified table compared to ELF). However, on Twizzler, this is more likely:

```
| Section | vaddr      | len   | offset | perms |
|---------|------------|-------|-------:|:-----:|
| .text   | 0x1000     | 0x800 |      0 |  r-x  |
| .rodata | 0x1800     | 0x100 |  0x800 |  r--  |
| .data   | 0x40001000 | 0x100 | 0x1000 |  rw-  |
| .bss    | 0x40001100 | 0x130 |    N/A |  rw-  |
```

Note the major change is just in the vaddr field, where we've bumped the data and bss sections to be loaded into the second object slot of the address space (object size 0x40000000). This is already how Twizzler operates. We just need to extend it to be a little more general for loading position-independent libraries. The above example loads data into object slots 0 and 1, respectively. It does this by first directly mapping the executable object to slot 0 (the linker script ensures this direct mapping is correct), and then creates a new data object and copies
the initial image of the data section from the executable (this is done via the copy-from primitive to leverage copy-on-write), which is then loaded into slot 1. For a position independent library, we'll allocate two consecutive slots and map the executable and read-only data into the first, and the writable data and bss into the second.

At this point, we need to run the standard dynamic linking algorithm, with some small exceptions, to relocate and link any loaded programs and libraries. Intra-compartment symbol resolution
results in standard dynamic library function calls, whereas inter-compartment results in the limitation of communication to secure gates. The main exceptions to the standard linking process are to ensure that allocations are performed intra-compartment by default, and to ensure that all calls stay within a compartment unless using a secure gate.

## Nandos

TODO

## More Details about the Irreducable Rust Runtime

Above we talked a bit about the stack and the heap, two of the most important parts of the runtime. But there is more.

### Thread Local Storage

The thread pointer, mentioned above, points to a per-thread data structure. The complication here is that we'll need to isolate compartments' TLS data from other compartments.
We can do this by using the higher-overhead TLS implementation that dynamic libraries use, where the thread pointer points to a vector of TLS regions, and we call a helper function
to actually load a reference to a TLS variable.

This vector then must be protected appropriately: read-only, except for the runtime. The runtime sets up the TLS vector, and then other libraries can use that to find their regions. On compartment switch, the runtime could change the thread pointer to a limited vector. This means that we'll need to protect updates to the thread pointer, which we will do by saying that
the thread pointer can be updated in contexts that have write access to the thread's repr object. A thread doesn't _need_ write access to its own repr, so we can prevent compartments from
changing the thread pointer. Finally, the kernel can verify properties of the thread pointer (not writable in source or target context) on compartment switch.

### Upcall pointer

This isn't an architectural pointer, but it is necessary for the kernel to deliver synchronous events to a thread. We can play the same trick: disallow updates to the upcall pointer, and have it set so that it enters the runtime on upcall.

### Unwinding

Since it's undefined behavior to unwind across an FFI boundary, there's a good chance we'll need to catch unwinding panics in a trampoline for a security gate. So if you write something like this:

```{rust}
#[security_gate]
pub fn foo(x: u32) -> u32 {...}
```

Under the hood, it'll get rewritten to something like:

```{rust}
pub fn foo(x: u32) -> core::thread::Result<u32> {
    core::panic::catch_unwind(|| __do_foo(x))
}

fn __do_foo(x: u32) -> u32 {...}
```

# Drawbacks
[drawbacks]: #drawbacks

The security limitation do add overhead on a security context switch. However, the comparison should not be to library calls, but to _invoking an entire new process_ on unix.

It is a huge undertaking. We could instead skip this, port a dynamic linker, etc. But I think that would miss out on an opportunity to leverage Twizzler's features to build a better
programming and system model.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This design builds directly off the Twizzler security model and implements, as far as I know, the simplest way to expose a secure programming interface to programmers such that
the secure programming is easy to do.

This RFC is careful to walk a line between being opinionated about how programming should be done on Twizzler and remaining sufficiently flexible (as would be expected by an OS).
However, it does essentially propose a _reference runtime_, which brings up worries about locking us into a particular ecosystem. However, the alternative is essentially _no_ standard
programming environment for Twizzler, which is unacceptable.

# Prior art
[prior-art]: #prior-art

Some useful papers and concepts:
* Lightweight Contexts (OSDI)
* Jails
* Solaris Doors
* Dynamic linking
* Compartmentalization

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What does the runtime trait look like?
- How do we hot-plug that runtime?
- What does the unix compatibility story look like?
- More syntax bikeshedding about the cli stuff.
- Key management...
- Can we return things using the stack in the shadow stack setup?
- Should the runtime be async by default?
- What does it look like to write a command line interface nando? Should it be a `#[cli_interface]` macro that we can use to auto-gen a CliCall struct that contains type info, etc?
- Can we implement this with ASLR?

# Future possibilities
[future-possibilities]: #future-possibilities

We can imagine the runtime providing deep introspection on the libraries and executables it loads and isolates. For example, imagine a debugger (controlled by the runtime) that can seamlessly transition from debugging a "script" in the REPL to debugging a loaded library (via step-in).

If all system software is written this way, and maintains OS configuration data and runtime telemetry data in objects, the REPL can expose a query-like interface for interacting with the OS, and can seamlessly be extended via nandos and secure gates.
