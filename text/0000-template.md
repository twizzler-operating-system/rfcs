- Feature Name: `kernel_rwlock`
- Start Date: 2022-08-31
- RFC PR: [twizzler-rfcs/rfcs#0000](https://github.com/twizzler-operating-system/rfcs/pull/0000)
- Twizzler Issue: [twizzler-operating-system/twizzler#0000](https://github.com/twizzler-operating-system/twizzler/issues/0000)

# Summary
[summary]: #summary

This RFC defines an interface that Twizzler kernel developers could use to implement a read-write
lock. The interface allows different implementations of a read-write lock to be used which may be suitable for different use cases.

# Motivation
[motivation]: #motivation

Reader-writer locks allow shared access to data, only requiring mutual exclusion for writes. 
Reader-writer locks, as opposed to a traditional mutex, does not have to pay the cost of 
serialized reads. A reader-writer lock is a good choice to protect infrequently updated state, 
such as a data structure that is read-mostly.

Consider for example [`TICK_SOURCES`](https://github.com/twizzler-operating-system/twizzler/blob/main/src/kernel/src/time.rs#L20) 
which is used to contain all clock sources in Twizzler. Most operations involve reading 
information about a clock source, such as its current value. The only time we see `TICK_SOURCES`
being modified is when we add a clock source in 
[`register_clock`](https://github.com/twizzler-operating-system/twizzler/blob/main/src/kernel/src/time.rs#L23-L30).
The problem with the implementation of `TICK_SOURCES` is that it uses a `Spinlock`, although 
thread-safe, reads not modifying data need to wait behind other readers. `TICK_SOURCES` would 
be able to more efficiently serve users if it allowed multiple readers to operate on it as so:

```rust
// initialize a rwlock in the unlocked state, and specify an implementation with ReadPrefer
pub static TICK_SOURCES: RwLock<ReadPrefer, Vec<Arc<dyn ClockHardware + Send + Sync>>> = RwLock::new(Vec::new());

pub fn register_clock<T>(clock: T)
where
    T: 'static + ClockHardware + Send + Sync,
{
    // specify when we want to write/modify the data
    let mut clock_list = TICK_SOURCES.write();
    let clk_id = clock_list.len(); // read
    let clk = Arc::new(clock);
    clock_list.push(clk.clone()); // write
    // ...
}

// only reading from TICK_SOURCES would look like this
// multiple threads can hold the lock to only read the data
{
    let clock_list = TICK_SOURCES.read();
    let num_clocks = clock_list.len() - CLOCK_OFFSET;
}
```

Twizzler has no implementation of a read-write lock. The expected outcome of this RFC is to
introduce a reader-writer lock in the kernel as well as to provide a standard interface which 
developers could use to implement different reader-writer lock mechanisms.

> This data structure should not be directly used by users. The `RwLock` interface is only meant
> to be used in the kernel. User programs should use what is in the standard library. 
> This RFC has no direct impact on user programs.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The main data structure that Twizzler developers interact with is `RwLock<R, T>`. This 
allows them to use different synchronization policies. To implement different synchronization policies
developers are going to interact with the `RawRwLock` trait. The goal of this type and interface
is to allow shared access to data while also allowing different implementations to be used for
different use cases. We rely on an external crate[^1] which defines these interfaces, but the implementation
is specific to Twizzler.

[^1]: For a more extensive explanation of these types and interfaces, see the 
    [documentation](https://docs.rs/lock_api/latest/lock_api/) for the `lock_api` crate.

## `RwLock<R, T>`

This is a type of lock that allows multiple readers and only a single writer at a time. Depending
on the implementation, readers or writers may starve waiting on each other. Other, more complex,
implementations may provide some fairness [[2]](#2).

`R` in `RwLock` indicates a type which implements a trait, allowing this type to use different
implementations. `T` is simply the data that this lock wraps around.

This type exposes a few methods, the basic ones are `read` and `write` indicating the semantic
meaning of the operation.
```rust
fn read(&self) -> RwLockReadGuard<'_, R, T>
fn write(&self) -> RwLockWriteGuard<'_, R, T>
```

Both methods return a lock guard which is used to manage a reference to the lock and allows 
holders of the lock to obtain the data by dereferencing those types. Another thing these guards
do is manage the lock state itself by releasing the lock when it is dropped.

## `RawRWLock`

This trait defines the underlying methods that `RwLock` will use to correctly manage lock state
and enforces the invariants that a reader-writer lock provides.

```rust
pub unsafe trait RawRwLock {
    type GuardMarker;

    const INIT: Self;

    fn lock_shared(&self);
    fn try_lock_shared(&self) -> bool;
    unsafe fn unlock_shared(&self);
    fn lock_exclusive(&self);
    fn try_lock_exclusive(&self) -> bool;
    unsafe fn unlock_exclusive(&self);

    fn is_locked(&self) -> bool { ... }
    fn is_locked_exclusive(&self) -> bool { ... }
}
```

The `shared` methods are when threads want to obtain shared access to the data the lock protects
as is the case for reads. The `exclusive` methods give exclusive access to the shared data which
is something that is needed for writes. `INIT` is simply a constant copy of the lock state when
initially created in the unlocked state. `GaurdMarker` is a marker type which determines if the
type implementing this interface is safe to `Send`.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

`RwLock` and it's trait interface `RawRwLock` arise from the need to provide efficient synchronization
for reads that do not need to be serialized, as well as the realization that different implementations
of `RwLock` are more efficient in different scenarios.

We initially began by taking inspiration from the Rust standard library's [`RwLock`](https://doc.rust-lang.org/std/sync/struct.RwLock.html).
Although it's interface made sense, it abstracts over a single implementation may not
be performant for special use cases. Then we sought to create our own type, much like the
standard library, except on in which we could specify the implementation using a type parameter.
The end result was satisfactory, however minimal and relied on an unstable feature. Then we came to learn of the 
[`lock_api`](https://docs.rs/lock_api/latest/lock_api/) crate which provides an interface similar
ours and is `no_std` compatible! 

Rather than re-invent the wheel, we choose to use their abstractions while the implementation
is specific to Twizzler and its use cases. This has a few benefits. The main one being that
we can focus on the implementation instead of the interface, and the other being that extending
the functionality of basic interface only requires implementing other traits exposed by the
library.

The integration of this crate into the Twizzler ecosystem would expose the appropriate types in
`src/kernel/src/rwlock.rs`. Take for example an implementation of a read-preferring lock.

```rust
use lock_api::{RawRwLock, RwLock, GuardSend};

use crate::{condvar::CondVar, spinlock::Spinlock};

// re-export types
pub use lock_api::{RwLock, RwLockWriteGuard, RwLockReadGuard};

// read-preferring lock state
pub struct ReadPrefer {
    cond: CondVar,
    inner: Spinlock<ReadPreferInner>
}

struct ReadPreferInner {
    active_readers: u64,
    active_writer: bool
}

// rwlock abstraction
unsafe impl RawMutex for RawSpinlock {
    type GuardMarker = GuardSend;

    const INIT: ReadPrefer = ReadPrefer {
        cond: CondVar::new(),
        inner: Spinlock::new(
            ReadPreferInner {
                active_readers: 0,
                active_writer: false
            }
        )
    };

    fn lock_shared(&self) {
        let mut lock_state = self.inner.lock();
        // increment number of readers so that others
        // writers know that we are waiting
        lock_state.active_readers += 1;
        // wait for any active modifications on data
        while lock_state.active_writer {
            lock_state = self.cond.wait(lock_state);
        }
        // continue on to read the data
        // lock on the lock_state is released on drop
    }
    
    fn try_lock_shared(&self) -> bool { todo!() }
    
    unsafe fn unlock_shared(&self) {
        let mut lock_state = self.inner.lock();
        // remove our presence
        lock_state.active_readers -= 1;
        // notify any others waiting on us
        // if we are the last ones out
        if lock_state.active_readers == 0 {
            self.cond.signal()
        }
        // lock on lock_state will unlock when fallen out of scope (Drop)
    }

    fn lock_exclusive(&self) {
        let mut lock_state = self.inner.lock();
        // wait while there are other readers reading the
        // data or other writers modifying it
        while lock_state.active_readers > 0 || lock_state.active_writer {
            lock_state = self.cond.wait(lock_state);
        }
        // we now hold the lock, time to set the flag
        // to let others know there is a writer modifying the data
        lock_state.active_writer = true
        // continue to write to the data
        // lock on lock_state will unlock when fallen out of scope (Drop)
    }
    
    fn try_lock_exclusive(&self) -> bool { todo!() }
    
    unsafe fn unlock_exclusive(&self) {
        let mut lock_state = self.inner.lock();
        // remove our presence
        lock_state.active_writer = false;
        // notify any others waiting on us
        self.cond.signal()
        // lock on lock_state will unlock when fallen out of scope (Drop)
    }

    fn is_locked(&self) -> bool { todo!() }
    fn is_locked_exclusive(&self) -> bool { todo!() }
}
```

For the sake of brevity we only show the implementations of the `lock`/`unlock` functions used by
`read`/`write` in `RwLock`. This implementation of `RawRwLock` lets readers immediately read the
data unless there is a writer actively modifying the data. Writers have to wait for other writers
to finish modifying the data or any readers waiting to read the data. This condition is ensured
by allowing readers to increment `active_readers` even before being able to actually read the data.
[`CondVar`](https://github.com/twizzler-operating-system/twizzler/blob/main/src/kernel/src/condvar.rs) 
and [`Spinlock`](https://github.com/twizzler-operating-system/twizzler/blob/main/src/kernel/src/spinlock.rs)
are a part of the Twizzler kernel.

A user of this lock policy could use it in a local or global context as so:

```rust
// global context
static DATA: RwLock<ReadPrefer, u64> = RwLock::new(42);

// in a local context
{
    let data: RwLock<ReadPrefer, u64> = RwLock::new(100);

    // let's read what's in there
    let value = {
        data.read();
        *data
        // lock unlocked at end of scope
    };

    assert!(value == 100u64);
    // now let's update it
    {
        let inner = data.write();
        // let's use some other data
        let val = DATA.read();
        *inner = *val
        // both locks dropped at end of scope
    }

    // what's inside now???
    assert!(*data.read() == 42u64);
}
```

# Drawbacks
[drawbacks]: #drawbacks

The drawbacks to this approach are the reliance on the `lock_api` crate and it's interfaces. 
Making changes to this interface would require upstreaming changes to the crate or making a fork
and using that instead. Alternatively defining our own front-end for `RwLock` or the use of extension
traits.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

This design is the best space of all possible designs since it defines a generic reader-writer
lock interface. It allows different implementations to be defined whilst operating behind the
same interface. The [Rust standard library](https://doc.rust-lang.org/std/sync/struct.RwLock.html), 
[FreeBSD](https://www.freebsd.org/cgi/man.cgi?query=rwlock&apropos=0&sektion=9&manpath=FreeBSD+13.1-stable&arch=default&format=html), 
and [Linux](https://elixir.bootlin.com/linux/latest/source/include/linux/rwlock.h) all expose 
a similar interface for their reader-writer locks, as does 
[`RwLock`/`RawRwLock`](#guide-level-explanation). However, they
are less general and only allow a single implementation to exist for all use cases.

The impact of not introducing a reader-writer lock would result in more contention on shared, 
read-mostly data structures.

# Prior art
[prior-art]: #prior-art

As stated before, the design for `RwLock` uses the interfaces defined in the `lock_api` crate.
The actual implementation for a reader-writer lock is up for debate since it varies on how
the shared data would be used. Rust's standard library 
[`RwLock`](https://doc.rust-lang.org/src/std/sys/unix/locks/futex_rwlock.rs.html) 
and FreeBSD's [`rwlock`](http://fxr.watson.org/fxr/source/kern/kern_rwlock.c) both 
implement a reader-writer lock using atomic variables and spinning. 
Linux's [`rwlock_t`](https://www.kernel.org/doc/html/latest/locking/locktypes.html#rwlock-t), 
depending on its preemption policy either implements the aforementioned design or allows waiting
threads to be preempted, and has a different implementation.

**Papers**

Here are a few papers that mention why reader-writer locks are a good idea. For a decent 
summary, see [[1]](#1) which contains a systematic study of reader-writer synchronization. 
Brandenburg and Anderson also have an earlier paper, just focusing on their phase-fair 
reader-writer lock [[2]](#2). Courtois et al. present examples of read-preferring and
write-preferring locks using semaphores [[3]](#3).

<a id="1">[1]</a> Brandenburg, Björn B., and James H. Anderson. "Spin-based reader-writer synchronization for multiprocessor real-time systems." Real-Time Systems 46.1 (2010): 25-87. https://www.cs.unc.edu/~anderson/papers/rtsj10-for-web.pdf

<a id="2">[2]</a> B. B. Brandenburg and J. H. Anderson, "Reader-Writer Synchronization for Shared-Memory Multiprocessor Real-Time Systems," 2009 21st Euromicro Conference on Real-Time Systems, 2009, pp. 184-193, doi: 10.1109/ECRTS.2009.14. https://www.cs.unc.edu/%7Eanderson/papers/ecrts09b.pdf

<a id="3">[3]</a> P. J. Courtois, F. Heymans, and D. L. Parnas. 1971. Concurrent control with “readers” and “writers”. Commun. ACM 14, 10 (Oct. 1971), 667–668. https://doi.org/10.1145/362759.362813

# Unresolved questions
[unresolved-questions]: #unresolved-questions

**Implementation**
- As part of the implementation process we hope to agree on what synchronization primitives are appropriate to build reader-writer locks.
- It is unclear if the kernel would want to use different types of reader-writer locks throughout
the code base, depending on the shared data, or if a more general strategy should be implemented
for all `RwLock`'s.

**Out of Scope**
- Testing the lock implementation for correctness is out of scope for this RFC. 
- It is unclear if a kernel profiling framework is required to test performance as well.

# Future possibilities
[future-possibilities]: #future-possibilities

A natural extension of this RFC is to implement other lock policies for the interfaces
exposed by `lock_api`. Another possible direction is to integrate the `lock_api` interfaces with existing locking code (e.g. `Spinlock`, `Mutex`).