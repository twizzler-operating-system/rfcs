- Feature Name: Glossary of Summer Work
- Start Date: 2024-06-24
- RFC PR: [twizzler-rfcs/rfcs#0009](https://github.com/twizzler-operating-system/rfcs/pull/0009)
- Twizzler Issue: [twizzler-operating-system/twizzler#0009](https://github.com/twizzler-operating-system/twizzler/issues/0009)

# Summary
[summary]: #summary

This is a glossary of some of the terms defined in the June 24 retreat, including Twizzler and its components, as well as some other projects being worked on by the lab.

# Motivation
[motivation]: #motivation

This glossary is meant as a quick reference guide, as well as an aid to intoducing newer lab members to the work quickly.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Twizzler
Twizzler is an incomplete Operating System. Its most notable traits are discussed in [this paper](https://www.usenix.org/system/files/atc20-bittman.pdf), though it has some undergone some changes since that paper was published.
Projects under the Twizzler name or supporting Twizzler include the following:
### File System
The File System is comprised of several projects:
- The Naming System (more info? ^^)
- A "Forgetting System" known as [Lethe](https://cs-people.bu.edu/dstara/pdfs/Lethe.pdf)  enables secure forgetting of encryption keys in a way that makes retrieval impossible.
- A Storage Driver (This does not exist and will need to be made soon!)
### CHERI
[CHERI](https://www.cl.cam.ac.uk/research/security/ctsrd/cheri/) is hardware that can support the unique file system that Twizzler has.
### MemOS in Twizzler
Twizzler needs more OS function. (More information needed? ^^)
### The Twizzler Kernel
![A diagram of the Twizzler Kernel.](/../assets/twz_kernel_diagram_black.svg)
#### App
An application written in Rust. This is written exactly like any other Rust program, but with a link to the Twizzler package. Drivers and the NVMe are considered applications in Twizzler's architecture.
#### Twizzler Runtime API
This manages object creation with Rust, among other things.
#### Twizzler-abi
A program that emulates Unix syscalls to work with the Twizzler kernel.

## USB Safety
USB flash drives aren't usually safe. We're trying to make a safe USB flash drive.
### "Sneakernet"
Sneakernets are data connections where the transfer involves physically moving storage media (such as a flash drive.) USB flash drives aren't usually secure beyond needing to have them to access them, and are vulnerable to:
- Being swapped with a lookalike (masquerading) (not our concern for the demo)
- Having their raw, unencrypted data read once they are obtained
### Getting rid of data
Currently, "secure deletion" is done by strapping thermite to the data media and physically destroying it. Using math would be better.

## Lethe
[Lethe](https://dl.acm.org/doi/pdf/10.1145/3578353.3589541) is a project concerning secure deletion of files. It does so by securely "forgetting" encryption keys, turning data that needs to stay hidden into gibberish.

## The Twisted Stick
The Twisted Stick will be a USB flash drive that is fully secure, with no way to read data that is stored on it, except by a verified user.
The Twisted Stick will run on [Twizzler]() and have the following structure of parts:
![Twisted Stick Diagram.](/../assets/twisted_stick_layout_black.svg)
### USB Stack
The API between the Stick and the client.
Connects to the client and the Twisted Service.
Currently being worked on by Michael and Thomas.
### Dynlink and Monitor
Connects to the kernel and the Twisted Service.
Currently being worked on by Daniel.
### Twisted Service
A connection between the USB Stack and the file system (including the pager and Lethe.)
Connects to the Dynlink and Monitor, the Pager, and Lethe.
Currently being worked on by Ananya and Clara.
### Lethe
See [Lethe](https://dl.acm.org/doi/pdf/10.1145/3578353.3589541).
Connects to the Pager and the NVMe.
Currently being worked on by Eugene and Zephiris.
### Twizzler Kernel
The kernel of the Twizzler operating system.
Connects to the Dynlink and Monitor and the Pager.
### NVMe
Non-Volatile Memory Express. A protocol to access the fancy flash memory.
### Flash
The flash far memory. Twizzler is built to optimize for its unique properties. (Is that correct? ^^)
### Security Monitor
There is a security monitor that enforces isolation between the above components.
### Goals with the Stick
Our intention is to implement an interface of Store, Retrieve, and Forget.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

N/A (This is the glossary)

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

N/A (This is the glossary)

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

N/A (This is the glossary)

# Prior art
[prior-art]: #prior-art

N/A (This is the glossary)

# Unresolved questions
[unresolved-questions]: #unresolved-questions

N/A (This is the glossary)

# Future possibilities
[future-possibilities]: #future-possibilities

This glossary may be updated in the future.
(Should this glossary be updated in the future?)
