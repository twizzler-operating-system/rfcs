# Twizzler Legacy Networking Stack

## Introduction

This document is intended to convey motivation, goals, and design of the Twizzler traditional networking stack, which is
intended to implement the commonly used networking protocols that allow for a Twizzler program to interact with other,
non-Twizzler machines. This is distinct from the planned “distributed Twizzler” work, which intends to provide a global
execution and data placement environment across a cluster of machines.

This traditional stack need not, at this time, implement a plethora of protocols. Since the first goal for this stack is
to implement enough of a traditional stack to interact with, say, a Linux server, we are only focused on implementing
Ethernet, NIC drivers, TCP, UDP, and IPv4 at this time. That said, we need to be careful not to build ourselves into a
corner if we choose to expand in the future.

## Terminology
This document discusses the interaction between a Twizzler service that implements a software-multiplexing traditional
networking stack (the **netmgr** or net manager) and a program that wants to interact with the network (a net client).
Unfortunately, the word “client” is a little overloaded here, sorry about that.

## Goals

 * Provide a mechanism for programs to interact with the network directly, that is, programs that want to do things like:
     * Open a TCP connection to a service on a server.
     * Send and receive UDP packets.
     * Listen for incoming TCP connections, and handle new connections.
     * Send and receive raw packets at the layer 3 level.
 * Allow efficient communication of packet data between the net client, the netmgr, and a NIC.
     * This requires buffer management on both sides of the connection between the net client and the netmgr, so that the net client can allocate and fill buffers without interacting with the netmgr (and vice versa).
 * We expect to provide a clean and ergonomic networking interface to the net client only at the level above the direct interaction between the net client and the netmgr.
 * Allow for efficient first-class asynchrony in the core interface.

# Technical Design

## Netmgr and net client shared objects

The netmgr and net client communicate through a collection of 4 objects, encapsulated in an nm_handle (or just, the
handle). The handle’s 4 objects are shared between the net client and the netmgr, and the form a two-way communication
pathway wherein the netmgr can send requests to the net client, and vice versa, where each request can be “completed”
(by sending a signal back) to indicate that a request has been completed (and thus, for example, all resources allocated
by the sender for that request can be released).

These 4 objects are: 
 * The transmission queue (Qt), used by the net client to send requests to the netmgr.
 * The receive queue (Qr), used by the netmgr to send requests to the net client.
 * The transmission buffer (Bt), which allows the net client to allocate buffers for sending to the netmgr.
 * The receive buffer (Br), which allows the netmgr to allocate buffers for sending to the net client.

The transmission objects pair up and are used to send packet data from the net client to the netmgr, and the receive
objects pair up and are used to send packet data from the netmgr to the net client. In general, to send packet data from
one side to the other, one would allocate a buffer from the appropriate object, fill that buffer, and then pass a
pointer to that buffer over the appropriate queue. This allows pass-by-reference packet data for both sides, reducing
copy overhead.

Note that these are buffers, and not packets necessarily. Nothing prevents using buffers to interact with a stream-type
connection, for example. That said, the buffers can also be used as packets, wherein a whole packet can be placed in a
buffer. Additionally, these buffers must be designed such that they can be sent directly to a NIC via DMA.

The queues provide an asynchronous communication protocol between the net client and the netmgr. Each queue is a pair of
multiple-producer/single-consumer queues that supports four (optionally non-blocking) operations:
 * submit – used to submit a request. 
 * receive – used to receive a request.
 * complete – used to complete a request. 
 * get_completed – used to read completions. 
A request is submitted from the submitter to the receiver, after which the receiver receives it. For
example:

```{rust}
struct Foo {
	x: u16
}

// fn submit<S>(queue: &Queue, id: u32, item: S)
queue.submit(0, Foo {x: 42});

// fn receive<S>(queue: &Queue) -> (u32, S)
let (id, foo) = queue.receive();
```

Of course, the receive and submit are done by different programs in this case (one is, say, the net client and the other is the netmgr). Following the receive, the receiver can do some work with the request. Once it has completed that work, it can send a completion notification back to the sender:

```{rust}
struct Bar {
	y: u32
}

// fn complete<C>(queue: &Queue, id: u32, item: C)
queue.complete(0, Bar {y: 42});

// fn get_completed<C>(queue: &Queue) -> (u32, C)
let (id, foo) = queue.get_completed();
```

The type of the completion notification can be different from that of the submitted request. However, the primary
contract here is that the u32 ID sent with the request is later used to indicate which request is completed. This is
made easier by the higher-level queue interface available in the twizzler-queue crate, which is used by the networking
interface. This boils down to interfaces: An interface that allows getting a request from the queue and later completing
it (both of which are asynchronous).  An interface used to submit a request and wait until we get a response (also is
async).

## Net client to netmgr protocol

The net client submits data of type TxRequest, and gets back completions of type TxCompletion:
```{rust}
enum TxRequest {
	Listen(ListenInfo),
	SendData(ConnectionId, PacketData),
    SendDataTo(ListenInfo, PacketData),
	Connect(ListenInfo),
	Shutdown(ConnectionId),
    NewHandle(ConnectionId),
	Close,
}

enum TxCompletion {
	Okay,
	NewConnection(ConnectionInfo),
	ListenReady(ConnectionId),
    NewHandleData(ConnectionId, NewHandleObjects),
	Error(TxError),
}
```

For now, let’s circle back to the question of configuring aspects of protocols, layers, and stateful connections later.

These enum variants allow for the net client to make requests to the netmgr, including setting up
listening posts, sending packet data with or without a stateful connection, connecting to a
listening post somewhere else, shutting down a connection, and disconnecting from the netmgr.

The netmgr submits requests to the net client with the following RxRequest, which are completed with
type RxCompletions:

```{rust}
enum RxRequest {
    RecvData(ConnectionId, PacketData),
    RecvDataFrom(ListenInfo, PacketData),
    Shutdown(ConnectionId, CloseInfo),
    NewConnection(ConnectionInfo),
    Close,
}

enum RxCompletion {
    Okay,
}
```

A couple general points:
 * The net client submits TxRequests using Qt. The netmgr listens for incoming requests on Qt.
 * The netmgr submits RxRequests using Qr. The net client listens for incoming requests on Qr.
 * The ConnectionId type is a simple integer value, intended to allow the net client and the netmgr
   manage stateful connections.
 * Both sides may issue a Close request, which, upon completion, indicates that neither side will
   use this handle for anything, and all underlying resources may be freed.
 * The net client may not respond with an error, as all requests from the netmgr must be handled
   with either success or total failure (the handle being closed).

### Setting up listening

A net client that wishes to listen to incoming connections first submits a `TxRequest::Listen`,
passing a listen info:

```{rust}
struct ListenInfo {
    addr: NodeAddress,
    service: ServiceAddress,
    ...,
}
```

This allows the netmgr to associate a particular listen point (address and port, for example) with
this handle (and thus, this net client). When the listening point is setup, the netmgr will complete
this request with `TxCompletion::ListenReady` (or an error), passing back a ConnectionId. This
ConnectionId will be used in future communications to indicate on which listen request incoming
connections are coming in.

When a new connection has been established, the netmgr can send information about the new connection
back to the net client by sending an `RxRequest::NewConnection`, passing back a ConnectionInfo:

```{rust}
struct ConnectionInfo {
    listen_id: ConnectionId,
    id: ConnectionId,
    addr: (NodeAddress, ServiceAddress),
    peer: (NodeAddress, ServiceAddress),
    ...,
}
```

From here, the ConnectionId in the id field can be used to send and receive data (see below). Each
side of the connection can send a `TxRequest::Shutdown` to their netmgr, indicating if they wish to
shutdown sending, receiving, or both. If a connection is shutdown in some way, the netmgr will send an
RxRequest::Shutdown to indicate the way in which the connection has been shutdown (receiving only,
sending only, or fully closed). When a connection is fully broken, the netmgr will send Shutdown
request to indicate the connection is fully shutdown, regardless of how it previously has indicated
shutdown to the net client for this connection. Shutdown can also be used to stop listening on a
listening point.

### Sending and receiving data on stateful connections

A net client may send data to the netmgr to be sent over a connection using `TxRequest::SendData`,
where it specifies a ConnectionId and some PacketData. For more discussion on PacketData, see below
(buffer management). The netmgr will then send the data in that buffer over the wire via the
protocols of the connection. This process is asynchronous and may take time, and thus the completion
notification may not be sent back immediately. Once the completion is sent back, the netmgr may not
use the buffer anymore, and thus the net client is free to free or reuse it.

A net client receives data by listening on a handle's Qr, and handling `TxRequest::RecvData`. The
RecvData request passes to the net client a ConnectionId to indicate which connection this data is
coming in on, along with the PacketData that has been received. Similar to SendData, the net client
can then send back a completion notification for this request to indicate that it has finished with
the data, and thus the netmgr can reused the buffers.

**NOTE**: Since completion notifications may not be sent bac immediately, if a net client is sending data across multiple
connections, it should not block the others on waiting for a particular completion for a particular
connection. Further, completion notifications are not guaranteed to be returned in the same order
that requests were submitted, and it depends on the protocol if SendData requests on a single
connection are not reordered.

### Buffer management

The goals of buffer management are to provide the net client with an efficient way of filling a
buffer with data to be passed to the netmgr for sending. They must also be efficient at holding small
and large amounts of data. Finally, the each buffer object is the source of buffer allocations for
either the net client or the netmgr, and they *cannot* allocate buffers from the other buffer
object.

To allocate a buffer, the buffer object provides an interface like:

```{rust}
fn allocate_buffer(buf: &BufferObject, len: usize) -> ManagedBuffer;
```

A program may allocate a buffer with a maximum size (len, or a pre-defined maximum size). The
returned ManagedBuffer provides an interface for filling the buffer with data:

```{rust}
// TODO
```

Finally, the ManagedBuffer implements `Into<PacketData>`, so it can be easily converted into a form
consumable by the TxRequest or RxRequest API. PacketData looks like:

```{rust}
struct PacketData {
    buffer_idx: u32,
}
```

When receving a request with a PacketData, it can be converted into a Buffer (which really just
contains a reference to a buffer in the appropriate buffer object) via an API in the handle:

```{rust}
fn get_incoming_buffer(handle: &NmHandle, pd: PacketData) -> Buffer;
```

**Do we need to save space for headers?**
Most network protocols preprend headers onto the packet, and thus a typical buffer management system
will allow for efficient prepending of data onto a packet as it moves through the layers. The
details of how this is done can be saved for later in this document, but there is one major point I
want to make now: the network protocol headers **cannot** be placed in the client-supplied buffer when
sending a packet. This is because we cannot prevent the net client from sending a packet and then
quickly mucking with the protocol headers between when the netmgr writes them and when it sends them
to a NIC. Thus a network send operation will require either a) sending two buffers to the NIC to
DMA, or b) copying data from a client-supplied buffer into an internal buffer where it can securely
write the headers. It's likely that both of these will be optimal in different scenarios, and so the
netmgr will need to include some heuristics about when to use a particular option.

**NOTE**: Since buffers can be reused, there may be a question of security (a program holding on to
a reference to an old buffer that it indicates it is done with, thus cauing it to be reused). While
this can happen, it's not a problem, since a net client can only do this for buffers it already has
access to, and those buffers come from the shared buffer objects that it will *always* have access
to.

