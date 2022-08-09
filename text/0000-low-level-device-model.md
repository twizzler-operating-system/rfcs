- Feature Name: low_level_device_model
- Start Date: 2022-07-14
- RFC PR: [twizzler-rfcs/rfcs#0000](https://github.com/twizzler-operating-system/rfcs/pull/0000)
- Twizzler Issue: [twizzler-operating-system/twizzler#0000](https://github.com/twizzler-operating-system/twizzler/issues/0000)

# Summary
[summary]: #summary

This RFC defines the interface that the kernel provides for userspace device management and driver
implementation. It provides a generic interface that allows for userspace access to
device memory and features through Twizzler objects, including memory-mapped IO (MMIO) registers,
interrupt handling, device event messages, and structured device trees.

> This low-level interface is not intended to be used by application programmers, and even driver
> programmers are expected to use higher-level abstractions for some aspects of their work (coming in
> a future RFC), though many aspects of the interface presented here will be used directly.

# Motivation
[motivation]: #motivation

Devices and drivers are, by and large, very complicated and (along with the surrounding
infrastructure) make up large parts of modern OS kernels. In a more microkernel fashion, we would
like to support userspace taking as much responsibility for device management and drivers as
possible. However, due to the aforementioned complexity, providing a generic interface that allows
for programming all possible devices is tricky -- we need to provide an interface that allows
generic access to device memory (and/or IO ports), interrupt progamming services, etc., all while
trying to minimize the responsibility of in-kernel code to handle these cases.

This RFC does not attempt to describe a higher-level, more convenient driver-programming API set;
that is planned for a future RFC. Instead, we will cover only the basic user-kernel interface for
device abstractions.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Twizzler organizes devices and busses into a tree structure, where devices are children of busses,
which are children of some root node (devices can have children too). These devices are each an
object that contains information about the device as well as a mechanism of programming device
interrupts. Each of these devices can also have a number of additional objects attached to it
(called sub-objects) that provide access to device MMIO registers and bus-specific device
information.

This low-level interface is not intended to be used by application programmers, and even driver
programmers are expected to use higher-level abstractions for some aspects of their work (coming in
a future RFC), though many aspects of the interface presented here will be used directly.

The meaning of "device" here is a little fluid, as it depends on the semantics of the hardware of
the system. The following are "devices" according to this model:
 - Any "pseudo-devices" that the kernel might create as an abstraction atop some internal mechanism.
 - Any busses that the kernel identifies (e.g. a PCIe segment).
 - Devices that sit on a bus that would be programmed as a device on a bus (e.g. present devices on
   a given bus:device.function on a PCIe segment, including bridges etc.).

## Example 

As an example, lets consider a PCIe segment with several devices on it, say a NIC at `0:1.0`
and an NVMe device at `0:2.0`. The resulting tree would be:

```
[Bus Root]
  [PCIe Segment 0]
    [0:1.0 (NIC)]
    [0:2.0 (NVMe)]
  [...additional PCIe segments...]
  [...additional busses...]
```

Say the NVMe device has a region of device memory for programming the controller's MMIO registers, a
region for the interrupt table,
and the PCIe bus provides an info sub-object that contains PCIe information. The device would have
sub-objects that describe the device, and sub-objects that map all the device's memory for access.

**Why is the PCIe _bus_ considered a _device_?** The philosophy here is that the PCIe bus is
programmed by accessing memory-mapped IO registers in device configuration space (including bridges,
etc.). Thus we can allow userspace to program it like it's a device that has sub-devices.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The core of the device model centers around the *device object*, a Twizzler object whose base is of
type `DeviceRepr`. Internally, these device objects are organized into a *Device Tree*, where each
device object may have up to 65Ki children. The root of this tree is special object called the Bus
Root, which has no data and only acts as the root of the device tree. Operations on a device object
are abstracted by the Device type defined in the twizzler-driver crate.

## Tree access

The `Device` type exposes a function, `children(&self)`, which returns an iterator that iterates
over all this device's children. This is a wrapper around a kaction (see: sys_kaction) request to the kernel on the
device object with the command `KactionCmd::Generic(KactionGenericCmd::GetChild(n))` where `n` is
the nth child of the device in the tree. The kernel returns either the object ID of the child
device, or `KactionError::NotFound` if `n` is larger than the number of children.

The tree is manipulated via bus-specific APIs. For example, the PCIe bus uses userspace to perform
device initialization, and thus exposes a kaction command for initializing a device on a given entry
of the PCIe segment. Busses that support device removal can also expose functions to remove devices
from the tree.

## The DeviceRepr type

The device object's base type is a DeviceRepr, which has the following layout:

```{rust}
#[repr(C)]
pub struct DeviceRepr {
    kso_hdr: KsoHdr,
    device_type: DeviceType,
    bus_type: BusType,
    device_id: DeviceId,
    interrupts: [DeviceInterrupt; NUM_DEVICE_INTERRUPTS],
    mailboxes: [AtomicU64; MailboxPriority::Num as usize],
}
```

The `DeviceType` enum can be either `Unknown` (and should be ignored), `Bus`, or `Device`. The
`BusType` enum specifies which bus this device is attached to (or, it it's a bus, what type of bus
this is). Finally, the last identifier is the `DeviceId`, which is a newtype wrapping a `u32` whose
meaning is bus specific.

In the example from earlier, the NIC would have a device ID that encodes 0:1.0 as a 32-bit integer,
bus type PCIe, and device type Device.

### Interrupts

Each device supports up to `NUM_DEVICE_INTERRUPTS` interrupt entries. The actual mechanism for
setting up interrupts with the kernel is bus-specific and relies on a small amount of in-kernel
bus-driver code. The `DeviceInterrupt` struct is as follows:

```{rust}
#[repr(C)]
pub struct DeviceInterrupt {
    sync: AtomicU64,
    vec: InterruptVector,
    flags: DeviceInterruptFlags,
    taken: AtomicU16,
}
```

When a registered interrupt fires, the kernel's interrupt handler writes a non-zero value to `sync`
and executes a thread_sync wake up on that word of memory, causing any user threads sleeping on that
variable to wake up. The `DeviceInterruptFlags` field a bitflags of size u16 and is currently
unused. The `vec` field refers to the actual vector number allocated by the kernel, and can be used
to program device interrupt funcionality (e.g. MSI or MSI-X in PCIe). Finally, the `taken` field is
used to indicate that a given entry is used or free.

### Mailboxes

A device has several mailbox entries. These are distinct from interrupts in that they are raised by
software (either in the kernel or not) and are statically initialized. Messages sent to device
mailboxes are not specified in this RFC.

The `DeviceRepr` has `MailboxPriority::Num` mailboxes, each a different priority level: High, Low,
and Idle. Each mailbox is a single `AtomicU64`. Submitting a message to the mailbox involves
performing a compare-and-swap (CAS) operation, writing a non-zero value if the value is zero. Should
the CAS fail, the submitter may perform a thread_sync sleep operation on the mailbox word. Should
the CAS succeed, the submitter must perform a thread_sync wake operation on the mailbox word.
Mailbox communication is not intended to be efficient.

When checking the mailbox, driver software performs an atomic swap with 0, reading the value. A
non-zero value means that the mailbox had a message. On swap returning non-zero, software must
perform a thread_sync wake up on the mailbox. Software may sleep on this word when receiving.

Driver software should prioritize the High priority mailbox above that of device interrupts, should
prioritize the low
priority mailbox below interrupts and High priority mailbox messages (but still ensure that the
mailbox gets checked even if the device is constantly sending interrupts), and should prioritize the
Idle mailbox lowest of all (and is allowed to ignore messages in this mailbox as it sees fit).

## Sub-objects

Each device has some number of sub-objects. Each sub-object can be one of the types specified by
`SubObjectType`: `Info` or `Mmio`. An info sub-object is an object whose base data is bus- or
device-specific. An mmio sub-object contains a header that describes the mapping, followed by mapped
MMIO space in the object space for accessing device memory. The Device type provides functions for accessing these sub-objects:

 - `fn get_mmio(&self, idx: u8) -> Option<MmioObject>`
 - `unsafe fn get_info<T>(&self, idx: u8) -> Option<InfoObject>`

The get_info function is unsafe because there is no checking that `T` is the correct type for the
stored data. The fact that the index is a u8 means that there can be up to 256 sub objects per type.

### MMIO Sub-Objects

The `MmioObject` type wraps the mmio sub object and allows for accessing the information about this
mapping and the mapping itself:

 - `fn get_info(&self) -> &MmioInfo`
 - `unsafe fn get_mmio_offset<T>(&self, offset: usize) -> &T`

The MmioInfo struct describes information about this mapping, including its length, a
device-specific `info` field, and the caching type of the mapping (uncachable, write-back, etc.).
The `get_mmio_offset` function returns a reference to a location within this mmio memory mapping.

## Example: PCIe

PCIe is a common bus mechanism. It provides access to devices via a series of segments, each of
which have a region of physical memory comprising its _configuration space_. Each device on the PCIe
segment is assigned a _bus_, _device_, and _function_ value that, together, indicate which page of
the configuration space is associated with this particular device (the device's PCIe configuration
registers). Each device on a PCIe bus has a set of six Base Address Registers (BARs) that describe the location
and length of mappings to the device's memory (e.g. MMIO registers, interrupt tables, etc.).
Finally, PCIe requires devices to support message-signaled interrupts (either via MSI or MSI-X).

On startup, the kernel enumerates the PCIe segments (e.g. via the ACPI tables) and creates a
bus-type device per segment. The segment's device has one info sub-object that specifies the
information the kernel knows about the segment (start and end bus numbers). The device also has an
mmio sub-object that maps the entire configuration space for the segment as Uncachable. Thus
userspace can get to all configuration memory for this segment.

The userspace device manager enumerates over all the PCIe segments, and for each one performs a
standard PCIe probing algorithm by reading the configuration memory. For each device that it finds,
it executes a kaction on the bus device object:
`KactionCmd::Specific(PcieKactionSpecific::RegisterDevice)` and passes as argument the bus, device
and function number encoded into a 64-bit value. The kernel then creates a new device entry as a
child of the bus with the following sub-objects:

 - Info sub-object: `PcieDeviceInfo` (which contains identifying information about the device).
 - Mmio sub-object: this device's portion of the PCIe configuration space. The `info` field in the
   `MmioInfo` struct is 0xff.
 - Mmio sub-object: one for each mapped BAR. The `info` field in the
   `MmioInfo` struct is the BAR number.

When allocating an interrupt, software is responsible for programming the MSI or MSI-X tables and
the device to generate a proper interrupt. To register a device interrupt with the kernel's PCIe
driver, the kaction-specific command `PcieKactionSpecific::AllocateInterrupt` can be used. The
64-bit argument is an encoding of which `DeviceInterrupt` in the `DeviceRepr` to use, and additional
flags about isolation.

This provides sufficient support for userspace to implement drivers for devices that allows
programming of:

 - Device interrupts via message-signaled interrupts.
 - Access to the MMIO registers for the device's PCIe configuration space.
 - Access to the MMIO registers for the device's BARs.
 - Access to any other device memory exposed via PCIe.

### The NVMe device from the example at the start

The NVMe device I have maps the controller MMIO registers at BAR0 and the MSI-X table at BAR4. Thus
its MMIO sub-objects would be:

 - 0: The PCIe configuration space, info = 0xff, uncachable.
 - 1: BAR0, info = 0, uncachable.
 - 2: BAR4, info = 4, uncachable.

# Drawbacks
[drawbacks]: #drawbacks

One major drawback is some additional complexity over in-kernel device management. Since we now need
to coordinate between userspace and the kernel for device management, it complicates the user-kernel
interface. However, the vast majority of users can ignore this.

By putting all driver software in userspace, we add overhead to interrupt handling, which now must
(in the upper half anyway) schedule a thread and context switch in the worst-case. However, we
expect driver software that is sensitive to this to implement at least partial polling for
performance, and the additional overhead is minor anyway compared to monolithic kernel designs that
still schedule threads to handle interrupts.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

A device model is a required part of the operating system. While the large-scale device model,
mechanisms, driver APIs, and management are larger in scope than this RFC, here we lay out a base
level of support for devices atop which we can build additional abstractions.

This model is designed to provide a flexible model that requires little kernel support. The device
tree model is a generic structuring of devices that can fit many different hardwares (they don't
have to use the tree model and can just be flat). The interrupt model provides an abstraction that
is based around how message-signaled interrupts are designed, which are commonly used. The
sub-object abstraction is designed to allow for numerous memory mappings for devices that have a lot
of them, and the info objects allow arbitrary information about devices to be encoded. Finally, the
mailbox system is designed for software to communcate with drivers (or between drivers, or drivers
and the bus) in a generic way.

An alternative mechanism would be to implement it all in the kernel and run driver software via
loadable modules. While this model doesn't preclude loadable modules, putting things in userspace
dramatically simplifies the kernel.

# Prior art
[prior-art]: #prior-art

- Microkernel OSes often place driver responsibilities into userspace.
- Monolithic kernels instead have drivers in the kernel, either compiled in or loaded as modules.
  These drivers enjoy higher privilege, but must rely on kernel infrastructure for programming.
- The descriptions of PCIe were based on the latest official PCIe specifications.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Is the interrupt mechanism sufficient (for now) to allow effective programming of devices?
- What is the correct security model to apply here? Probably a set a approved programs can access
  the device tree and that's it.
- Some PCIe devices should have their memory mapped as write-combinting (framebuffers, for example),
  but PCIe doesn't encode this information in the BARs. We need a way for userspace to perhaps remap
  a BAR as WC.

# Future possibilities
[future-possibilities]: #future-possibilities

One major aspect that has been left out is some kind of generic "reporting" facility, where a driver
could report device status back to the device manager (or the kernel) some generic status
information. I have hesitated to design and include this, since I do not yet know what this would be
used for or look like, and would prefer instead to save it for a future RFC.

