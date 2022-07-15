- Feature Name: driver_model
- Start Date: 2022-07-15
- RFC PR: [twizzler-rfcs/rfcs#0000](https://github.com/twizzler-operating-system/rfcs/pull/0000)
- Twizzler Issue: [twizzler-operating-system/twizzler#0000](https://github.com/twizzler-operating-system/twizzler/issues/0000)

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



Explain the proposal as if it was already included in the system and you were teaching it to another Twizzler programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Twizzler programmers should *think* about the feature, and how it should impact the way they use Twizzler. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Twizzler programmers and new Twizzler programmers.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

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

- id system
- async that return a future in requester submit
- submit and submit for response

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
