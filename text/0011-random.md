- Feature Name: Source of Entropy
- Start Date: 2024-07-15
- RFC PR: [twizzler-rfcs/rfcs#0011](https://github.com/twizzler-operating-system/rfcs/pull/0009)
- Twizzler Issue: [twizzler-operating-system/twizzler#0011](https://github.com/twizzler-operating-system/twizzler/issues/0009)

# Summary

[summary]: #summary

Adds a secure source of entropy to the kernel, similar to /dev/random on Twizzler. It should seed itself at startup with a true random number generator (TRNG) and then use a Cryptographically secure Pseudo Random Number Generator (CSPRNG) to generate following requests for randomness.

# Motivation

[motivation]: #motivation

This RFC is useful because randomness is used for cryptographic operations and the kernel certainly requires cryptographic operations such as private/public key generation and UUID generation for the twizzler security model. It is also very likely userspace programs will also require entropy for userspace cryptographic operations (or games or other applications which require randomness).

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

The module should expose a `getrandom(dest: &mut [u8])` function (and syscall) which will entirely fill the `dest` buffer with bytes. It might block while doing so during the initial seeding of the CSPRNG. I also propose providing a `getrandom_nonblocking(dest: &mut [u8]) -> usize` which will do the same but without blocking, returning the number of bytes overwritten into `dest` (which can be 0).

### Example 1 (getrandom)

```rs
let mut buf = [u8; 32] // 256 bytes total
getrandom(&mut buf); // should fully overwrite random bytes into buf
```

### Example 2

```rs
let mut buf = [u8; 32] // 256 bytes total
// should write `bytes_overwritten` number of bytes into the
// buf. Might be 0 bytes
let bytes_overwritten = getrandom_nonblocking(&mut buf);
```

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

My current plan is to use the current kernel's `time` interface to measure CPU jitter (see [JitterRng](https://docs.rs/rand_jitter/latest/rand_jitter/struct.JitterRng.html)) to seed an entropy pool.

I intend to design an "EntropyPool" struct which is responsible for XORing bytes on top of the pool's current state while keeping track of the bytes of entropy each source provides. It should then periodically seed and reseed a CSPRNG (probably ChaCha12) once there is enough entropy gathered within the entropy buffer.

The CSPRNG should be the final source of entropy which will be returned via the above system calls. Before the CSPRNG is fully seeded calls `getrandom()` should block until it is seeded and calls to `getrandom_nonblocking()` should return 0.

# Drawbacks

[drawbacks]: #drawbacks

Why should we _not_ do this?

Maintaining a pool of entropy (and a CSPRNG) requires some additional memory and CPU footprint. In addition, users might end up running their own CSPRNG seeded off of our pool, which might make the kernel's redundant. That said, since the kernel itself requires cryptographically secure RNG for object management and permissions I think it's more than worth exposing the same source of randomness into userspace, just so that userspace doesn't have to reimplement a source of entropy. Also all other common OS's have a source of randomness available.

# Rationale and alternatives

[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art

[prior-art]: #prior-art

### Firsthand OS Documentation and Source Code:

- Linux tends to restore an entropy buffer via various sources of hardware randomness throughout the runtime of the device. Linux uses ChaCha20 for their CSRNG sourceas of version 4.8 [source](https://words.filippo.io/dispatches/linux-csprng/), which as of 4.8, makes /dev/random non-blocking except for at boot, bringing Linux's /dev/random implementation in line with FreeBSD's.
- FreeBSD (and variants) block at boot while gathering entropy to seed their CSRNG (FreeBSD has used Fortuna since 2014) and then reseed the CSRNG over time as more entropy is collected.
- MacOS and iOS uses the Fortuna algorithm as well (since 2019) for their CSRNG and seeds both at boot time and during runtime as well.
- Windows 10+ uses a pretty complex system, which is written about in detail [here](https://download.microsoft.com/download/1/c/9/1c9813b8-089c-4fef-b2ad-ad80e79403ba/Whitepaper%20-%20The%20Windows%2010%20random%20number%20generation%20infrastructure.pdf). In my opinion it's overkill to have a per-process CSPRNG (especially when most performance-critical applications which require a lot of randomness will bundle their own CSPRNG). However, it is interesting that they primarily use interrupt timing to append into their entropy pools which could be a good source for us to use as well.
- Android just uses the linux implementation and therefore also uses ChaCha20 for their random number generation.
- All OS's listed above use various means of gathering real-world randomness, and for future reference see [Linux's random](https://github.com/torvalds/linux/tree/master/drivers/char/hw_random), [CheriBSD's random](https://github.com/CTSRD-CHERI/cheribsd/tree/main/sys/dev/random) (which is based on [FreeBSD's random](https://github.com/freebsd/freebsd-src/tree/main/sys/dev/random)), and [MacOS's docs on their random number generation](https://support.apple.com/guide/security/random-number-generation-seca0c73a75b/web) (which is also based on FreeBSD's implementation).
- [JitterRng](https://crates.io/crates/rand_jitter) is a random number generator based on measuring timing of operations on the cpu, inspired by [HAVEGE](https://www.irisa.fr/caps/projects/hipsor/publications/havege-tomacs.pdf).
- OsRng depends on [getrandom](https://crates.io/crates/getrandom)

### Secondary Analyses

- [Blog post](https://words.filippo.io/dispatches/linux-csprng/) explaining the changes made to Linux's random.
- [GitHub issue](https://github.com/rust-random/rand/issues/699) discussing the quality of randomness of JitterRNG, a TRNG which is based on [HAVEGE](https://www.irisa.fr/caps/projects/hipsor/publications/havege-tomacs.pdf). Conclusion seems to be that JitterRng is mostly a last-resort.
- [Stack overflow](https://stackoverflow.com/a/74484189) answer on using RAM as a source of entropy. The other answers might be of use as well.
- Cryptoanalysis papers on ChaCha:
  - [Tertiary Review of ChaCha Exploits](https://ieeexplore.ieee.org/document/9766147) - concludes that [this paper](https://link.springer.com/chapter/10.1007/978-3-030-56877-1_12) is the best currently known key-recovery attack with time complexity $2^{230.86}$ for 7 round ChaCha. As ChaCha12 uses 12 rounds, it seems the 12 round version is secure (at least for now).

# Unresolved questions

[unresolved-questions]: #unresolved-questions

### To be resolved during the RFC process:

- I hope to also research the security difference between ChaCha12, ChaCha20, and Fortuna, although I suspect ChaCha12 is the better choice for us given that it is faster when there is no sha256 hardware implementation (which is the case on ARM), Linux chose ChaCha20 in 2016, and [this GitHub discussion](https://github.com/rust-random/rand/issues/932) as to why ChaCha20 is overkill (during RFC process before merging).

### During Implementation

- I hope to look into how many bits of entropy are considered to be "enough" to seed the CSPRNG as defined by the other operating systems.
- I hope to look into how many bits of entropy various sources provide.

### Out of Scope

- Eventually we should support more sources of randomness. At this point however we can start with an insecure timing-jitter based approach.

# Future possibilities

[future-possibilities]: #future-possibilities

### More Entropy Sources and Userspace Entropy Contributions

In the future we should add more sources of randomness as we add support for peripheral devices. Additionally we should support some standard and secure way to provide entropy to the kernel from userspace.

In FreeBSD this can only be provided by super-users due to security concerns of intentionally reducing entropy within the generator's internal state.

In Linux anyone can contribute additional entropy to the entropy pool, but the entropy count will stay the same.

I am not sure how we want to use Twizzler's permission system in determining who can contribute to the entropy pool. Ideally there should be some way to have drivers contribute to the pool so that interfaces to peripheral sources of entropy do not have to be added to the kernel.

### Other CSPRNGs

Generally speaking it is a good idea to keep track of the security of whichever underlying CSPRNG we are using just in case it ends up getting broken. For example FreeBSD (and along with it MacOS) previously used Yarrow before upgrading to Fortuna.
