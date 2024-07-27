- Feature Name: Source of Entropy
- Start Date: 2024-07-15
- RFC PR: [twizzler-rfcs/rfcs#0012](https://github.com/twizzler-operating-system/rfcs/pull/0011)

# Summary

[summary]: #summary

Adds a secure source of entropy to the kernel, similar to /dev/random on Twizzler. It should seed itself at startup with several true random number generators (TRNG) and then use a Cryptographically secure Pseudo Random Number Generator (CSPRNG) to generate following requests for randomness.

# Motivation

[motivation]: #motivation

The kernel requires cryptographic operations such as private/public key generation and UUID generation for the twizzler security model. These sort of cryptographic operations require a cryptographically secure source of randomness to ensure their security. It is also very likely userspace programs will also require entropy for userspace cryptographic operations (or games or other programs which require randomness).

# Guide-level explanation

[guide-level-explanation]: #guide-level-explanation

## Syscall API

The module should expose a `getrandom(dest: &mut [u8], flags: GetRandomFlags) -> usize` function (and syscall) which will entirely fill the `dest` buffer with bytes and returns the number of bytes written into the destination buffer. It might block while doing so during the initial seeding of the CSPRNG. `GetRandomFlags` for now will only consist of a single flag, NONBLOCKING, which will ensure `getrandom` does not block. Note that before the CSPRNG is seeded `getrandom` might write 0 bytes into dest and accordingly return a 0.

### Example 1 (getrandom)

```rs
let mut buf = [u8; 32] // 256 bytes total
let bytes_written = getrandom(&mut buf, 0); // should fully overwrite random bytes into buf
assert!(256 == bytes_written)
```

### Example 2 (nonblocking)

```rs
let mut buf = [u8; 32] // 256 bytes total
// should write `bytes_overwritten` number of bytes into the buf. Might be 0 bytes
let bytes_written = getrandom(&mut buf, GetRandomFlags::NONBLOCKING);
```

## Inner-Kernel API

The module should have a public function called `contribute_entropy(entropy: &[u8])` which will allow sporatic entropy contributions from sources which are not consistently able to provide entropy upon requrest. The module should also describe an `EntropySource` trait for sources which can consistently provide entropy upon demand:

```rs
trait EntropySource {
  fn try_fill_entropy(&mut self, dest: &mut [u8]) -> Result<(), rand_core::Error>;
}
```

There will also be a public registering function `register_entropy_source(source: dyn EntropySource)`. Registered entropy sources will be called during a reseeding of the CSPRNG.

### Example 1

```rs
let t1 = now()
// do something that takes a variable amount of time, preferably involving hardware
let t2 = now()
let timingDiff = t2 - t1
// only the first LSB because we mainly care about microsecond jitter
// as upper bits are typically 0 since the hardware action doesn't take that long
contribute_entropy([timingDiff as u8])
```

### Example 2

```rs
struct CpuEntropy;

impl EntropySource for CpuEntropy {
    fn try_fill_entropy(&mut self, dest: &mut [u8]) -> Result<(), rand_core::Error> {
        let mut dest_iter = dest.iter_mut();
        let mut rndrs_iter = self.try_iter()?;
        for (d, r) in dest_iter.zip(rndrs_iter) {
            *d = r?
        }
        Ok(())
    }
}

pub fn maybe_add_cpu_entropy_source() {
    if let Some(cpu_entropy) = CpuEntropy::new() {
        register_entropy_source(cpu_entropy)
    }
}

```

###

# Reference-level explanation

[reference-level-explanation]: #reference-level-explanation

My current plan is to use the current kernel's `time` interface to measure CPU jitter (see [JitterRng](https://docs.rs/rand_jitter/latest/rand_jitter/struct.JitterRng.html)) to seed an entropy pool.

I intend to design an "EntropyPool" struct which is responsible for XORing an array of hashed bytes on top of the pool's current state once there is enough entropy gathered in the pool to warrant a reseed. See the following quote from [_Cryptography Engineering_](https://www.schneier.com/wp-content/uploads/2015/12/fortuna.pdf) for justificationßß:

> making any kind of estimate of the amount of entropy is extremely difficult, if not impossible. It depends heavily on how much the attacker knows or can know [...]. It tries to measure the entropy of a source using an entropy estimator, and such an estimator is impossible to get right for all situations.

Rather than keeping track of entropy estimates, Fortuna, Yarrow's successor used by MacOS and FreeBSD, simply requires a seed of a certain total length rather than trying to estimate entropy amounts of each source since these entropy estimates are usually innacurate. It also saves previous entropy into a persistent file to be the startup seed for the next boot cycle to avoid blocking at boot time. For our implementation we will simply always block at boot to gather entropy, falling back to JitterRng if there is not enough entropy gathered within the time alotted.

It should then periodically seed and reseed a CSPRNG (probably ChaCha12) once there is enough entropy gathered within the entropy buffer.

The CSPRNG should be the final source of entropy which will be returned via the above system calls. Before the CSPRNG is seeded calls `getrandom()` should block until it is seeded (or return 0 if the NONBLOCKING flag is set).

# Drawbacks

[drawbacks]: #drawbacks

Why should we _not_ do this?

Maintaining a pool of entropy (and a CSPRNG) requires some additional memory and CPU footprint. In addition, users might end up running their own CSPRNG seeded off of our pool, which might make the kernel's CSPRNG redundant. That said, since the kernel itself requires cryptographically secure RNG for object management and permissions. I think it's more than worth exposing the same source of randomness into userspace, just so that userspace doesn't have to reimplement a source of entropy. Also all other common modern OS's have a source of cryptographically secure randomness available to userspace.

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

- I hope to also research the security difference between ChaCha12, ChaCha20, and Fortuna, although I suspect ChaCha12 is the better choice for us given that it is faster when there is no sha256 hardware implementation (which is the case on ARM), Linux chose ChaCha20 in 2016, and [this GitHub discussion](https://github.com/rust-random/rand/issues/932) as to why ChaCha20 is overkill. Edit: turns out FreeBSD also uses ChaCha20 as the hashing mechanism within Fortuna ([source](https://github.com/freebsd/freebsd-src/blob/main/sys/dev/random/fortuna.c#L525-L668)). So with all of that evidence, I think using ChaCha12 will be plenty okay for our purposes.

### During Implementation

- I hope to look into how many bits of entropy are considered to be "enough" to seed the CSPRNG as defined by the other operating systems.
- I hope to look into how many bits of entropy various sources provide.

### Out of Scope

- Eventually we should support more sources of randomness. At this point however we can start with an imperfect cpu jitter timing based approach with the possible addition of timing OS interrupts.

# Future possibilities

[future-possibilities]: #future-possibilities

### More Entropy Sources and Userspace Entropy Contributions

In the future we should add more sources of randomness as we add support for peripheral devices. Additionally we should support some standard and secure way to provide entropy to the kernel from userspace.

In FreeBSD this can only be provided by super-users due to security concerns of intentionally reducing entropy within the generator's internal state.

In Linux anyone can contribute additional entropy to the entropy pool, but the entropy count will stay the same.

The random module should eventually have a Twizzler entropy object with a write capability which the kernel is able to grant to various other userspace processes (such as the pager). Then those processes can pull entropy from the various drivers it relies upon (such as the NVMe driver in the pager's case).

### Kernelspace Entropy Contributions

The kernel might also be able to contribute entropy directly from interrupt timing.

### Other CSPRNGs

Generally speaking it is a good idea to keep track of the security of whichever underlying CSPRNG we are using just in case it ends up getting broken. For example FreeBSD (and along with it MacOS) previously used Yarrow before upgrading to Fortuna.
