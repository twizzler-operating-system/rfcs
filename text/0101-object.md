# Basic Objects 

Objects in Twizzler form the basis for memory organization and data identity in a global address
space. The core features that all objects share are:

 * Invariant references. Pointers in an object can reference any other object regardless of location.
 * Base structure. Objects have a "known entry point" that represents the overall type of the
   object.
 * Metadata. Objects have a metadata structure that contains a core metadata.
 * Access control. Objects may be read/write/execute depending on security context.
 * Metadata extensions. An object may "respond" to various APIs.

This document serves to outline the core object layout and the design and rationale of the
twizzler-object crate.

NOTE: This document is still in draft form, and as such many APIs are not finished. I would love feedback!

## Object Layout

Objects form a 1GB contiguous sparsely populated collection of pages, and are all identified by a
unique 128-bit ID. Objects may be either mutable or immutable (this property is, itself,
immutable)[^1]. An object may be either **shared** or **volatile**. A shared object is one that may
be operated on by multiple *instances of a host*[^2]. 

The high-level layout is:

```
+---+--------------------------+---+-------+
| N | B                      F | M | E     |
| U | A -->              <-- O | E | X --> |
| L | S -->              <-- T | T | T --> |
| L | E                      e | A | s     |
+---+--------------------------+---+---+---+
```

At the very start of an object, we devote an entire page to an unmapped page. This serves two
purposes:

 * Invariant pointers can be NULL and always refer to an unmapped page.
 * An unmapped page between objects prevents runaway writes from escaping an object.

After the null page we have the base of the object, which acts as a type for the object. The base
can be `unit` if there is no meaningful base type. The rationale for having a base type is that it
provides a simple way to represent a simple type and an entry point for objects. It allows the
kernel to interact with some kinds of objects without needing to understand the more complex aspects
of object layout.


After the base, the object memory is
application-defined up until the end of the foreign object table (FOT), which grows downward. Often the remaining memory
after the base is given to a per-object allocator. Near the top of the object, there is a metadata
struct that has the same layout for all objects. Following the metadata struct is a number of
extensions that allow the object to specify that is responds to a certain API.

The metadata region is needed because we need a mechanism for invariant pointers (the FOT), so it's
not that the overhead of the metadata is justified, it's that it's required. Even for objects with
no outgoing pointers, the metadata struct provides information for object ID derivation, base type
type verification, and commonly used meta extensions.

[^1]: Note that there's a chicken and egg problem here. An immutable object is immutable upon
    creation, and so it can only be created via the object create system call (which supports
    scatter-gather specification for how to construct object memory).

[^2]: An instance of a host includes 1) different computers, 2) different power cycles of a single
    computer, or 3) different kernels running on different heterogeneous devices in a single
    computer. Another way to think about it is that volatile objects fate-share with the running
    kernel they are associated with. Note that this doesn't mean that volatile objects are limited
    to a single host, but that they must *move* instead of being shared.

### Meta types

The meta struct is defined as follows:

```{rust}
struct MetaData {
    nonce: Nonce,
    kuid: ObjID,
    p_flags: u32,
    flags: u32,
    fotentries: u32,
    metaexts: u32,
    basetag: u64,
    version: u64,
}
```

The nonce, kuid, and p_flags are related to security, the flags are used for future extension, the
fotentries and metaexts fields count the number of FOT entries and meta extensions. The basetag and
version fields are used for BaseType versioning and verification.

The FOT starts just below the meta data struct, and is an array of entries defined as follows:

```{rust}
struct FOTEntry {
    union {
        id: ObjID,
        nameinfo: { name: u64, resolver: u64 },
    },
    refs: u32, // ref count of this entry
    flags: u32, // requested protections
    resv0: u32, // unused (must be zero)
    resv1: u32, // unused (must be zero)
}
```

The meta extensions entries are defined as:

```{rust}
struct MetaExt {
    tag: u64,
    value: u64,
}
```

The tag field specifies which extension this is, and the value field is dependent on the extension.
Tags have a simple constraint that if the top 32 bits are zero, then that is a value reserved for
the system.

## The twizzler-object Crate

The twizzler-object crate provides a foundation and interfaces for interacting with Twizzler
objects. It exports a selection of core types and traits that make it possible to build a
higher-level management system (if required):

 * The metadata types defined above (may be reexported from the twizzler-abi crate).
 * Object creation, deletion, and lifetime controls.
 * The `Object<T>` type.
 * Invariant pointers.
 * Object Safety trait.
 * Transaction handle trait.
 * Transaction-based interior mutability.

A metaphor for this crate is Rust's futures and async programming, which provides a set of traits
and framework for a runtime to be implemented.

### Object Safety

The crate provides an auto marker trait: ObjSafe. The ObjSafe trait denotes two things:

 * The data structure does not contain a non-invariant pointer.
 * The data structure ensures that mutation is possible only via transactions.

The first is done by `impl<T> !ObjSafe for *const T` (etc), and the second is done by implementing
`!ObjSafe` for things like UnsafeCell. Any memory that is located in an object should implement
ObjSafe (which happens automatically usually). Of course, a data structure may unsafely implement
ObjSafe.

### The BaseType trait

Object base types have a few constraints on top of simple object safety. They must be able to prove
that some persistent value stored in memory is a valid instance of the type without any provenance,
and they must provide an initialization mechanism for creating objects. The trait contains the
following:

```{rust}
trait BaseType {
    /// Constructs a Self, optionally using some arguments.
    fn init<T>(args: T) -> Self;
    /// List of all tag, version pairs supported by this type.
    fn tags() -> &'static [(Tag, Version)];
}
```

A BaseType's init function is called by the object creation function as a way to create the initial
object's base. An additional mechanism for creating an object by running an arbitrary transaction
will also be supported.

The tags and version information is used as a runtime check for `Object<T: BaseType>::base()`, to
ensure that an object actually has a `T` at its base. The version information is currently matched
against, but unused, with the purpose of later implementing an upgrade mechanism.

### TxCell

Rust's ideas of interior mutability map quite well to a transactional update mechanism. We can
support some code like this:

```{rust}
struct Foo {
    a: u32,
    b: u32,
}

let obj: Object<Foo> = ...;

transaction(|tx| {
    let base = obj.base_mut(tx);
    base.a = 42;
    Ok(())
});
```

But, as written (and say we were implementing the transaction as an undo log), we would need to
consider the entire Foo as possibly being modified, not just `base.a`. Thus we provide a
transaction-safe interior mutability type, TxCell, which provides:

```{rust}
fn get_mut(&self, tx: impl TxHandle) -> &mut T;
fn get(&self, tx: impl TxHandle) -> &T;
fn get_notx(&self) -> &T; // also implemented via Borrow?
```

Thus, it can only be called in a transaction context, and only exposes the inner owned value mutably
inside a transaction. Outside of a transaction, the TxCell can provide an immutable reference to the
interior value (this may require kernel thread synchronization, and will be discussed in a future
RFC).

We might also implement async versions of these.

### TxHandle

A TxHandle is an interface to a transaction implementation.

```{rust}
trait TxHandle {
    fn tx_cell_get(&self, cell: &TxCell) -> Result<...>;
    fn tx_cell_get_mut(&self, cell: &TxCell) -> Result<...>;
    fn base(&self, &obj) -> Result<...>;
    ...
}
```

### `Object<T>`

The key type the twizzler-object crate provides is the `Object<T>` type, which represents a single
Twizzler object. This type exposes the following interfaces:

```{rust}
fn id(&self) -> ObjID;
fn init_id(id: ObjID, prot: Protections, flags: InitFlags) -> Result<Self, InitError>;
fn base(&self);
// functions to add, get, and remove metaexts
// functions to view, add, and remove FOT entries (unsafe?).
// functions to tie and delete objects?
...
```

### Standard Meta Extensions

Twizzler reserves meta extensions with a tag value that has the top 32 bits as all zero for system
use and for standard universal extensions. Two system use values here are:

 * null (tag = 0x0): This marks the end of the metaext array (which may be before the end as
   specified by `MetaData::metaexts`).
 * tombstone (tag = 0xffffffff): A previous entry that has been deleted, and should be ignored. It
   may be reused, and it does not mark the end of the array.

#### Size (tag value: 0x1)

Some objects may have a notion of "size", where the amount of data in the data region grows and
shrinks such that there is always a maximal size (eg. file size in Unix). The value part of the
extension contains this size value. The API will be explored in a future RFC.

### Invariant Pointers

An invariant pointer functions similar to a raw pointer in semantics. It does not convey lifetime or
reference counting, and may be null.

```{rust}
#[repr(transparent)]
struct InvPtr<T> {
    p: u64,
    _pd: PhantomData<T>,
}

impl<T: ObjSafe> InvPtr<T> {
    /// Resolve this pointer using this object. Checks if the pointer is within the object.
    fn lea_obj(&self, obj: &Object<_>) -> EffAddr<T>;
    /// Resolve this pointer.
    fn lea(&self, obj: &Object<_>) -> EffAddr<T>;
    /// Move a pointer.
    fn moveptr(&mut self, tx: &TxHandle, other: &mut Self) -> Result<...>;
    /// Copy a pointer.
    fn copyptr(&mut self, tx: &TxHandle, other: &Self) -> Result<...>;
    /// Store a pointer.
    fn store(&mut self, tx: &TxHandle, other: &T) -> Result<...>;
}

impl<T> InvPtr<T> {
    fn null() -> Self;
    fn is_null(&self) -> bool;
    fn raw_parts(&self) -> (usize, u64);
}
```

The `EffAddr` struct can be dereferenced and contains a reference to an internal object handle for
the target object that contains the referenced data. This can be consumed or cloned into an
`Object<T>` that can be used. The `lea_obj` is an optimization.

## Alternatives

The design of invariant pointers forms the basis for sharing and the global address space. Other
invariant designs (fat pointers) have problems.

The choice to use this crate as an interface definition is based around avoiding being prescriptive
regarding how applications use objects. Instead, this crate only implements the bare minimum for
building transactions that can localize their effects on top of a notion of safety for storing a
type in persistent object memory. One could imagine, however, an alternative scenario where we instead build a full transaction
mechanism into the core of the twizzler object system.

## Status

This document is a draft and must be completed. Initial exploratory work has begun in the
`dbittman-object` branch on the twizzler repository.

## Future

Completion of this document requires:

 * Completed API specifications for the types outlined above.
 * Agreement on interface types.
 * Implementation notes on temporary object handles and FOT caching.

Future, planned RFCs include:
 * Access control and the Rust-based object model. How does TxCell handle the case where we can only
   read an object and we are calling `get`?
 * Names in FOT entries, and manual FOT entry specification.
 * Allocation and InvBox
 * The Pool abstraction, example transactions, InvRc.
 * Async extensions to the Tx APIs in this document.
 * Append-type objects and the Size meta extension.

