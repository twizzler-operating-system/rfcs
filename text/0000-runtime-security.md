- Feature Name: runtime_security
- Start Date: 2024-02-22
- RFC PR: [twizzler-rfcs/rfcs#0000](https://github.com/twizzler-operating-system/rfcs/pull/0000)
- Twizzler Issue: [twizzler-operating-system/twizzler#0000](https://github.com/twizzler-operating-system/twizzler/issues/0000)

# Summary
[summary]: #summary

Since Twizzler is focused so heavily around persistent state, managing runtime state can be tricky. In particular, there isn't yet a good story
for the interaction of security policy and runtime state objects (e.g. stack, heap).
This RFC extends the Twizzler security model with two key mechanisms: runtime object security, and security context instancing. The purpose is
to support _runtime objects_ (that is, objects created solely for runtime use, e.g. heap and stack objects). Such objects need to be easy and fast
to create, yet still must be subject to the same (or, as we'll see, nearly the same) rules for security of Twizzler objects.

# Motivation
[motivation]: #motivation

Components of the Twizzler system often need to create purely runtime objects. Applications that expect their heap data to be volatile and private
should be able to rely on that assumption (privacy and security from isolation). Further, applications that need to create these volatile workspaces
should not need to engage with the more complex security mechanism that enables security for persistent objects. Yet we still want to be able to express
fine-grained control over access policy for such objects. This RFC describes a set of minimal extensions to the existing security model that will enable
efficient use of runtime objects while allowing those objects to be subject to all security context rules as any other objects. In the common case, however,
runtime objects need not be referenced by security contexts for their use. Note that this is a vital property: the (persistent) security context information
need not contain references to (volatile) runtime objects, which is especially important when we consider that security contexts should be able to globally deny
access to objects that aren't needed.

Concretely, this enables easy use of the following patterns:

 - Private heap: one compartment has rw- permissions on an object, and no other compartment does. The compartment's security context need not be modified to include this object.
 - Shared heap: one compartment has rw- permissions on an object, while all other compartments can read it.
 - Stacks and TLS, same as above.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Security Context Instances

A security context is an object that contains data that the kernel can use to program the MMU to align with security policy. However, it's likely we'll want to support
there being multiple "instances" of these security contexts being active at once (e.g. a program tied to a security context S may be invoked twice, each in a separate compartment).
To support this, the kernel assigns an object handle (similar to how virtual address spaces work) when attaching a security context:

```{rust}
fn sys_attach(sctx: ObjID) -> Result<ObjID, ...>;
```

This is a purely runtime construct, the resulting object handle is a volatile object.

## Runtime Objects

When creating an object in Twizzler, one piece of data passed to the kernel's create_object syscall is "keyid". For standard objects, the keyid refers to the 
object ID of the object's associated public key, which can be used to verify signed capabilities for objects. For runtime objects, we instead store the object ID
of a _security context instance_, and set a bit in the object's metadata flags. When the kernel calculates permissions for a runtime object inside a given context,
if that object refers to that context in keyid, we first calculate the permissions from the security context (it may still reference it), including global masks and
object default permissions. After that, we augment the resulting permissions with "local default" permissions, stored next to the object's default permissions inside
the object's metadata. These permissions are not masked.

To maintain security, we require the following:

 - A runtime object must be volatile.
 - A runtime object with keyid S may only be created in a thread with security context S active (i.e. runtime objects for a given context may only be created within that context).

One exception is the monitor can always create runtime objects for arbitrary security contexts. This is necessary to bootstrap, and the monitor is already inside the TCB.

## Examples

Let's look at the shared heap example. A compartment, named C, creates a new runtime object, H, with keyid = C, local default permissions rw-, and global default permissions r--.
When code in compartment C accesses this object, the kernel will derive permissions rw-. When any other compartment accesses it, it will get permissions r--. Without this mechanism,
we would need to construct a capability for object H and write it into C's security context. This would be expensive and dangerous (could leave a dangling reference to a volatile object).

The private heap example is even simpler, it just sets global default to ---. Similarly, however, this would require expensive modification to C unless we have runtime object features as outlined in this RFC.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This doesn't require deep modifications to the system yet, since most of the security context implementation is pending anyway. But it will entail:

 - Modify the kernel's understanding of security contexts to use handles instead of security context objects directly.
 - Extend the monitor to handle security context handles.
 - Add a control bit in object metadata, and extend the default permissions field to include default permissions for runtime objects.

# Drawbacks
[drawbacks]: #drawbacks

- Requires a breaking change to object metadata, but that isn't stable yet anyway.
- Adds an arguable "escape hatch" via the monitor. I don't really agree, but we can argue about it.
- Conflates keyid as either a reference to a public key object OR a security context.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

An alternative is to support just instancing, and make it so that a compartment is two security contexts, a "runtime" context and a persistent one.
I find this to be just as complex, if not moreso, than this RFC's proposal. Further, it doesn't address the overhead issues.

# Prior art
[prior-art]: #prior-art

TODO

# Unresolved questions
[unresolved-questions]: #unresolved-questions

TODO

# Future possibilities
[future-possibilities]: #future-possibilities

TODO
