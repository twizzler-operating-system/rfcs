- Feature Name: driver_model
- Start Date: 2022-07-15
- RFC PR: [twizzler-rfcs/rfcs#0005](https://github.com/twizzler-operating-system/rfcs/pull/5)
- Twizzler Issue: [twizzler-operating-system/twizzler#0083](https://github.com/twizzler-operating-system/twizzler/issues/83)

# Summary
[summary]: #summary

This RFC extends RFCxxxx with a set of higher-level abstractions that drivers can use to program
devices. It describes centralized abstractions surrounding event streams, request-response queues, asynchrony,
and interrupts that will be provided by the twizzler-driver crate.

# Motivation
[motivation]: #motivation

Writing device drivers is hard, in no small part due to complex models inherent to the asynchronous
nature of interacting with the devices. Developers can make use of infrastructure surrounding device
drivers in most operating systems to lessen this complexity. Devices can be (often, largely) modeled
as separate computing devices that a) produce events for host software to consume, and b) receive
and respond to commands sent by host software. Thus we want to provide a mechanism that allows these
abstractions to be implemented by a majority of device drivers easily while providing a unified
framework that higher-level aspects of a driver can use.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The twizzler-driver crate provides a high-level wrapper around a collection of abstractions designed
to manage a single device (for definition of "device" and the `Device` type, see RFCxxxx), called a
`DeviceController`. When taking control of a device, driver software can create a `DeviceController`
(heretoafter referred to as "the controller") from a `Device`. After this point, the controller
manages the device and exposes several abstractions above the device:

 - A `DeviceEventStream`, which provides a way to get a stream of events, including mailbox events
   and device interrupts (see RFCxxxx).
 - A `DmaAllocator` (details for which will appear in a future RFC).
 - A `RequestEngineManager`, which provides a way to submit requests to the device and
   asynchronously await responses.

## Device Event Stream

The device event stream provides an async interface for handling (higher-half) interrupts and
mailbox messages. This means that some async executor is required; if this is too much overhead for
the application, direct access to the underlying Device allows for lower-level interrupt handling
without the async overhead.

This allows driver software to handle incoming events from the device. For example, a NIC that
receives a packet transfers the packet data to a receive buffer and then submits an interrupt.
Driver software will then see this interrupt via the event stream and can handle it appropriately.

The `DeviceEventStream` provides an interface for allocating an interrupt, which return an
`InterruptInfo`. Internally, the twizzler-driver crate handles the allocation of the interrupt
according to the transport mechanism (e.g. PCIe MSI/MSI-X) and registration with the kernel. The
`InterruptInfo` type exposes an async function `next` which returns the non-zero u64 value of the
next interrupt that fires or None if the interrupt is removed. Thus driver software can handle a
stream of interrupts as follows:

```{rust}
let int = controller.events().allocate_interrupt(...).unwrap();
while let Some(x) = int.next().await {
    while still_work_to_do() {
        // handle interrupt
    }
}
```

The `DeviceEventStream` also provides a method for accessing mailboxes, which can be used as
follows:

```{rust}
loop {
    handle_mailbox_message(controller.events().next_msg(MailboxPriority::High).await);
}
```

As in RFCxxxx, it is up to driver software to ensure correct prioritization of mailbox receivers.

## Request Engine Manager

The other main abstraction is to model the device such that it receives requests and asynchronously
responds. For example, submitting a packet for transmit to a NIC happens by the driver software
constructing an entry in a transmit queue and then notifying the device that the queue has been
appended to. After the device submits the packet, it responds by indicating that that queue entry
has been consumed and sends an interrupt. Note that since this often requires interrupt, this
abstraction sits logically above the event stream abstraction.

Driver software can create a new `Requester` through the controller, which acts as a manager for
inflight requests. The requester has a generic type that implements `RequestDriver`, which is a
trait that implements device-specific logic for submitting a request. 

 - `fn shutdown(&self)` -- shuts down this requester, notifying the driver, and cancels any
   in-flight requests.
 - `fn is_shutdown(&self) -> bool` -- returns true if the requester is shutdown.
 - `async fn submit(&self, requests: &mut [SubmitRequest<Driver::Request>]) ->`
       `Result<InFlightFuture<Driver::Response>, SubmitError<Driver::SubmitError>>` -- submits a
       number of requests to the device via the driver, and return a future for when the requests
       are completed.
 - `async fn submit_for_response(&self, requests: &mut [SubmitRequest<Driver::Request>]) ->`
       `Result<InFlightFutureWithResponses<Driver::Response>, SubmitError<Driver::SubmitError>>` --
       same as submit but the output of the future contains a vector of responses (one for each
       request).
 - `fn finish(&self, responses: &[ResponseInfo<Driver::Response>])` -- called by the driver to
   indicate which requests have completed.

The `Driver` in the API above is the generic in `Requester<Driver: RequestDriver>`. This trait is as
follows:

```{rust}
#[async_trait::async_trait]
pub trait RequestDriver {
    type Request: Copy + Send;
    type Response: Copy + Send;
    type SubmitError;
    async fn submit(&self, reqs: &[SubmitRequest<Self::Request>]) -> Result<(), Self::SubmitError>;
    fn flush(&self);
    const NUM_IDS: usize;
}
```

Note: async functions in traits is unsupported by Rust at time of writing, so we use the async_trait
crate here to make it possible to write this.

The associated types `Request`, `Response`, and `SubmitError` refer to the types of objects that this requester
will submit to the device, responses the device sends back, and errors that the request driver can
generate when submitting requests.

The `SubmitRequest` type is a wrapper around the driver's Request associated type that includes an ID
of type u64. The number of inflight requests will not exceed `NUM_IDS`, and all IDs will be less
than that value (this allows the driver to specify backpressure and queue limits). Similarly, the
`ResponseInfo` type is a wrapper around the `Response` type that also includes the ID of the request
that is associated with this response, and a boolean indicating if this response is considered an
error or not.

### Example

Driver software is responsible for implementing the RequestDriver. As an example, say we have a NIC
that has a transmit queue for packets. We submit to the transmit queue by writing entries starting at the
head and then writing an MMIO register to indicate that the head has moved. Say the queue is 128
entries long. When a queue entry has been handled, the NIC writes back to the queue entry a status
word and sends an interrupt to indicate a new tail position. The implementation of the RequestDriver would:

 - Set NUM_IDS to 128.
 - Define Request to be whatever a transmit queue entry looks like.
 - Define Response to be the status word.
 - Define SubmitError to be an enum of possible submit errors.
 - Implement submit such that it:
   - Keeps a map of queue position to `SubmitRequest` ID.
   - Submits the requests. If it cannot submit them all, it internally
     asynchronously awaits until it can.
   - Optionally writes the new head to the head MMIO register.
 - Implement flush to write the head register to the MMIO register.
 - Implement an interrupt handler that runs when this queue's interrupt handling routine should be
   signaled. This routine goes through the queue starting from the old tail to the new tail and
   reads all the status words for those entries, constructing an array of `ResponseInfo` types,
   eventually calling `finish` on the requester. This routine may need to wake up the submit
   function after it reads out the status words and records the new queue tail.

Another example we can consider is an NVMe driver, which differs from the NIC driver above by having
a separate completion queue. This possibility is the reason behind the abstracted request IDs --
this lets the driver and requester handle out-of-order responses.

### Usage

The requester uses this implementation to expose the interface above that lets higher-level driver
software interact with the device via this async request-response API. For example, a caller could
do the following:

```{rust}
let req = create_packet_tx_req();
let mut sreq = SubmitRequest::new(req);

let fut = requester.submit(&mut [sreq]);
let res = fut.await.unwrap(); // awaits until the requests are submitted (may be a SubmitError).
let res2 = res.await; // awaits until the device responds to the requests.
match res2 {
    Done => {...} // all requests were handled, none error'd.
    Shutdown => {...} // the requester shutdown while requests were in-flight.
    Errors(x: usize) => {...} // all requests were handled, at least one error'd, all error occur after position x.
}
```

Use of the `submit_for_response` function looks similar to the above, except `res2` could also be
matched to `Responses(Vec<Driver::Response>)`. The order of responses matches the order of requests.
The reason that you have to issue two `await`s is that one is for having submitted all requests, and
the second is for all requests being responded to.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

These abstractions will be implemented by the twizzler-driver crate, and will be optional features
to allow for driver software that does not need them.

## Events

The device event stream is largely a straight-forward wrapper around the lower level device
interrupt and mailbox mechanisms, presenting them as an asynchronous stream instead of something
more akin to a condition variable. The future returned by `InterruptInfo`'s next function uses the
twizzler-async crate along with the interrupt definitions discussed in RFCxxxx to construct a
blocking future that returns the next interrupt.

The interrupt allocation routine operates with some bus-specific knowledge to properly allocate an
interrupt for the device before constructing an `InterruptInfo`. On drop, the `InterruptInfo` calls
back into the event stream manager to deallocate the interrupt, thus tying the lifetime of the
interrupt to the `InterruptInfo` struct.

The mailbox message system internally keeps a queue of messages that haven't been handled. This is
so the event stream can receive the mailbox message and clear up the mailbox for reuse even if no
thread has called `next_msg`. These queues should have a maximum size, causing old messages to be
dropped if necessary. The `next_msg` function:

 - Checks the mailboxes of all priorities, enqueuing any messages found there.
 - Dequeues messages from the highest priority queue that has messages, if the queue is at least as
   high priority as `min`. Messages here are returned immediately.
 - If no messages are present, blocks this async task on the arrival of new messages.

Note: since High priority mailbox messages take priority over interrupts, interrupt next() functions
will also check the High priority mailbox, enqueuing if a message is found.

## Requester

The requester internally keeps track of:

 - Inflight requests.
 - Available request IDs.

### Request IDs

Request IDs are a simple unique identifier for requests. When calling submit, the caller passes a
slice of SubmitRequests, which internally contain an ID. The caller, however, is not responsible for
assigning IDs -- that is handled internally (hence why it's a mutable slice).

The available request IDs are managed by an AsyncIdAllocator, which exposes three functions (note:
this is all completely invisible to the user):

 - `fn try_next(&self) -> Option<u64>`
 - `async fn next(&self) -> u64`
 - `fn release(&self, id: u64)`

Both try_next and next get an available ID, but next asynchronously waits until one is available.
Two adjacent calls to any next functions are not guaranteed to return adjacent ID numbers.

### Inflight Request Management

The submit function returns an InFlightFuture, and the submit_for_requests function returns an
InFlightFutureWithResponses. These each output a SubmitSummary and a SubmitSummaryWithResponses when
awaited. Each of these futures internally hold a reference to an InFlightInner that is shared with
the requester, and manages the state for the inflight requests.

The InFlightInner contains:

 - A waker for the task that is awaiting the future.
 - An `Option<SubmitSummary>` for returning to the awaiter when it's ready, and additional state to
   construct this value.
    - e.g. a map of request IDs to indexes in the submit slice (may be unused if not tracking
      responses).
    - e.g. a vector of responses that gets filled out as responses come in (may be unused).
 - a count, counting how many requests have been responded to.

The requester internally keeps a map of request IDs to InFlightInner so that it can match a response
with an inflight that manages it.

Finally, the requester's shutdown function shall ensure that, after internally recording the
shutdown status so that future calls will fail, it drains all internal InFlightInners and fills out
their ready values to indicate shutdown, after which it wakes any waiting tasks.

# Drawbacks
[drawbacks]: #drawbacks

One major drawback to this design is overhead. Using these abstractions requires pulling in the
twizzler-async runtime _and_ tolerating the (small) overhead of async for interacting with the
device. This may be hard to tolerate in embedded environments (async executor size may be too large)
and extreme high performance environments (async overhead)[^1].

A counter argument here would be that these abstractions are optional, but the counter-counter
argument could be that the convenience they offer may make them non-optional in practice, where most
drivers use them, requiring their use even in embedded systems. However, in systems that are truly
space-limited, one is often working within the confines of an exact hardware set, so manual
implementation of small drivers is likely regardless, and the larger drivers used on server machines
will not be applicable.

[^1]: This argument is less convincing to me. The interrupt handling routines, for example, only
    incur the async overhead when they are out of work. Polling or delaying calling the next()
    functions can nearly completely mitigate this overhead.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The main purpose here is to provide a common framework for common aspects of driver development,
thus accelerating the creation of device driver software. By defining this RequestDriver trait, we
allow driver software to be split into two parts: the lower half that submits requests on a queue,
and the higher half, which submits requests via the Requester interface, simplifying driver software
by handling the complexities of async and out-of-order responses. The use of async here is
especially important, as it allows driver software to just submit requests and await responses.

Several rationales:

- The request ID system is designed to allow the driver to internally manage its understanding of
  IDs and how they relate to the device queue(s). This makes it possible for the requester to
  internally manage in-flight state independent of the driver, and handle async and out of order
  responses.
- There is a separate flush function to allow for enqueuing a number of requests without incurring
  the overhead of actually notifying the device.
- The submit and submit_for_responses functions are async and both return another future, meaning
  one needs two awaits to get the SubmitSummary. This is because we separate the successful
  _submission_ of requests from the completion. Imagine we want to submit 200 requests to a queue
  that has 128 entries. We'll have to wait at some point. Thus we allow the caller to await full
  submission and then later await responses if it likes (or it can drop the future and not get any
  responses or information).
- We separate submit and submit_for_response because collating the responses has additional
  overhead, and a given submit may not care about the actual responses. Thus we provide an option
  for just submitting and awaiting completion without recording responses.

# Prior art
[prior-art]: #prior-art

- TODO

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- One unresolved question is in the ergonomics of building a driver that uses all these types that
  reference each other. I plan to dogfood this interface by way of an NVMe driver.
- The DmaAllocator is a major component of drivers, allowing the driver to talk about physical
  memory. That will be discussed in a future RFC.

# Future possibilities
[future-possibilities]: #future-possibilities

- TODO
