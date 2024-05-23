---
title: "Media over QUIC - TransFork"
abbrev: "moqtf"
category: info

docname: draft-lcurley-moq-transfork-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
v: 3
area: wit
workgroup: moq

author:
 -
    fullname: Luke Curley
    organization: Discord
    email: kixelated@gmail.com

normative:

informative:
  moqt: I-D.ietf-moq-transport

--- abstract

TODO Abstract


--- middle

# Fork

This draft is based on moq-transport-03 [moqt].
The concepts, motivations, and terminology are very similar on purpose.

I absolutely believe in the motivation and potential of Media over QUIC.
The layering is phenomenal and addresses many of the problems with current live media protocols.
I fully support the goals of the working group and the IETF process.

However, there are some flaws with MoqTransport that I'd like to address.
It's been years and we're still unable to align on the most critical property of the transport... how to utilize QUIC.
The draft compromises by supporting multiple different approaches, but it does by leaving important properties undefined.
In our RUSH to standardize a protocol, the QUICR solutions have led to WARP in ideals.

This fork is meant to be a constructive, alternative vision.
I recognize that some of these ideas may he flawed, but my hope is that this draft can start a conversation on how to improve MoqTransport.


## Critique
Before diving into the differences, I want to highlight the major issues with MoqTransport and why a fork is necessary.

### Object Model
MoqTransport introduces the concept of Tracks, Groups, and Objects.
While not explicitly stated, these implicitly map to media concepts:

- **Track**: Audio/Video Track
- **Group**: Video Group of Pictures
- **Object**: Audio/Video Frame

But how do you deliver these over QUIC?
Unfortunately, "it depends".

The application is responsible for choosing the properties of the object model via the "delivery preference", determining how the object model is mapped to QUIC streams (or datagrams).
The application is further responsible for determining how to prioritize/drop objects or introduce gaps.

This was always meant to be a compromise, allowing for experimentation without nailing any of the protocol down.
However, we've managed to define so many optional properties that there is nothing concrete to build upon.
The stream mapping conflicts with the object model, which only serves to give the same name to wildly different "objects".
Every statement has to be prefixed with: "when using a stream per X".

The object model should have clearly defined properties.
A developer should be able to understand the properties of a "group" without caveats or undefined behavior.

### Non-Reference Frames
While not explicitly stated, the complexity in MoqTransport stems almost entirely from a single use-case: the ability to drop individual non-reference frames.

Video codecs involve complicated dependencies between frames/slices.
A non-reference frame is a leaf node in the dependency tree, often a B-frame in H.264.
The idea is that non-reference frame can be dropped during congestion, allowing the decoder to continue with the remaining frames with no problem.

This idea sounds good on paper but it's a trap.
Good encoders try to avoid non-reference frames because they fundamentally reduce the compression ratio;
dead-end bits that can't be used for prediction.
You gain the ability to halve the frame rate during congestion in order to drop 20% of the bitrate.
However, the cost is a net 10% increase in bitrate and additional latency.

But the real problem is that this requirement is pandora's box to implement.
QUIC streams are reliable/ordered by design, so in order to drop individual frames it becomes necessary to subdivide.
The complex dependency graph is not written on the wire, so frames are delivered and decoded based on crude priorities. 
It's very east for a relays or decoder to inadvertently drop/expire reference frames and present an undecodable mess to the viewer.

MoqTransfork makes the ability to drop individual frames a non-goal.
It relies instead on decode order within a GoP, an over-simplification that avoids artifacts and is used by WebRTC, HLS/DASH, and RTMP alike.
An advanced application that wants to drop non-reference frames can still use multiple tracks to form layers, via SVC or B-pyramids.


## Differences
Here's a high level overview of the differences between MoqTransport and MoqTransfork.

### QUIC Mapping
The MoqTransfork object model is static and maps directly to QUIC streams.

The data within a QUIC stream is delivered reliably and in order until a STREAM_FIN, or abruptly truncated with a RESET_STREAM.
Most importantly for live media, QUIC streams can be independently transmitted, prioritized, and dropped.

MoqTransfork builds on top of these properties as each Object is a QUIC stream.
Due to the nature of QUIC, most operations are done at the stream/object level, such as dropping, reordering, and prioritizing.
It's not possible to circumvent QUIC stream properties (ex. gaps in a stream) and attempting to do so defeats the purpose of using QUIC.

The result is transport with well-defined properties.
It's significantly simpler while still providing the necessary flexibility for a wide range of applications.

### Subscriber Priority
As the original proponent of producer choosen priorities (send order), I'm ashamed to admit that I was wrong.
Reality is more naunced; both the subscriber and publisher need to work together.

MoqTransfork delegates the priority decision for the last mile to the subscriber.
This is done via a `priority` (between tracks) and `order` (within a track) field within `SUBSCRIBE`.
If there's a single viewer, then this priority can be used all the way to origin.

However, when relays and multiple viewers are involved, there needs to be some form of a tiebreaker.
The publisher indicates the default value via `SUBSCRIBE_OK` or `INFO` and the relay should use it during conflicts.
The result is that the last hop uses the viewer's preference, but upstream hops could instead use the producer's preference.

### Byte Offsets
When a connection or subscription is severed, it's desirable to resume where it left off.

MoqTransport implements this via subscriptions that can start/end at an object ID within a group.
This is an okay approach, however it can result in redownloading objects (ex. incomplete keyframes) especially when groups are delivered out of order.
But critically it will cause a fragmented cache, as a relay needs to parse incoming streams, split them at object boundaries, and operate independently on each object.

MoqTransfork instead utilizes FETCH to serve incomplete OBJECTs at a byte offset, relying on the fact that QUIC streams are tail dropped when reset.
This subtle change dramatically simplifies relays, allowing them to stream from a cache by byte offset instead of a parsed marker (object ID).
An advanced relay can now even forward STREAM chunks out of order, which was not always possible with Object IDs.

### Control Streams
MoqTransport control messages are fine, but have the potential for head-of-line blocking as they share a single stream.
There's also just a lot of messages to transition state machines, such as UNSUBSCRIBE, SUBSCRIBE_ERROR, and SUBSCRIBE_DONE.

MoqTransfork continues the trend of leveraging QUIC with a separate control stream per transaction.
There are no more messages that start with `UN` or end with `_DONE`, `_ERROR`, `_RESET`, etc; the stream state machine is used instead.
This simplifies the protocol since an error code is often sufficient.

Additionally, flow control can be used to limit the maximum number of bidirectional streams per direction, removing the need for something like `MAX_ANNOUNCE` or `MAX_SUBSCRIBE`.

### No Datagrams
MoqTransport supports sending objects as QUIC datagrams via a track preference.
This is often useful when the desired latency is below the RTT as it avoids some overhead.

The problem with QUIC datagrams is that they do a poor job catering to higher latency subscribers.
When retransmissions are possible, such as when RTT is low and bandwidth is available, the one-and-done nature of datagrams results in a worse user experience.
It might make sense to use datagrams for a live stream, but not if you want to serve those same objects for the VOD.

The MoqTransfork alternative is to create a QUIC stream, write the data, and send a RESET_STREAM after the TTL (unless acknowledged).
This is a few more bytes on the wire but gains flow control, drop notifications, and (potential) retransmissions.

I think datagrams are an optimization that can be added later.
They would be separate from Objects and applicable only for unreliable live scenarios.
But QUIC datagrams have a lot of pitfalls and are eeirily simimilar to QUIC STREAM frames.

# Concepts
Many of the concepts are borrowed from MoqTransport so I'm not going to rehash them here.
A future draft will.

## Object Model
The MoqTransfork object model consists of:

- **Broadcast**: A collection of Tracks from a single producer.
- **Track**: A series of Objects within a Broadcast.
- **Object**: A series of Frames within a Track, served via a QUIC stream.
- **Frame**: A sized payload of bytes within a Group.

### Broadcast
Each broadcast is identified by a unique name within the session.
A publisher may optionally ANNOUNCE a broadcast or the subscriber can determine the name out-of-band.

The MoqTransport draft refers to this as "track namespace".
I couldn't help but bikeshed.

### Track
Each subscription is scoped to a single track and starts/ends at an Object boundary.
The subscriber chooses the priority/order of each track, dictating which track arrives first during congestion.

Each track has a unique name within the broadcast.
There is currently no way to discover tracks within a broadcast; it must be negotiated out-of-band.
For example, a `catalog` track that lists all other tracks.

### Object
Just like QUIC streams, an Object is delivered reliably in order with no gaps.
There is a header containing any stream-level properties.

Both the publisher and subscriber can cancel an OBJECT stream at any point with QUIC's `RESET_STREAM` or `STOP_SENDING` respectively.
When an Object is cancelled, the publisher transmits an OBJECT_DROPPED message to inform the subscriber.
A subscriber may issue a FETCH to resume at a given byte offset.

### Frame
Frames currently only provides framing, hence the name.

This is useful in some applications but can also be redundant or increase memory usage, as the size of a frame must be known upfront.
This may be removed in a future version of the draft.


# Control Messages
Bidirectional streams are used for control messages.

Note that QUIC bidirectional streams have both a send and recv direction can be closed or reset (with an error code) independently.
This is used to indicate completion or errors respectively.

The first varint of each stream indicates the type.
Streams may only be created by the indicated role, otherwise the session MUST be closed with a ROLE_VIOLATION.
A stream type MUST NOT contain messages defined for other stream types unless otherwise specified (ex. INFO).

|------|-----------|------------|
| ID   | Type      | Role       |
|-----:|:----------|------------|
| 0x0  | Session   | Client     |
|------|-----------|------------|
| 0x1  | Announce  | Publisher  |
|------|-----------|------------|
| 0x2  | Subscribe | Subscriber |
|------|-----------|------------|
| 0x3  | Fetch     | Subscriber |
|------|-----------|------------|
| 0x4  | Info      | Subscriber |
|------|-----------|------------|


## Session Stream
The Session stream contains all messages that are session level.
The client MUST open only one Session stream immediately after establishing the QUIC/WebTransport session.

The client sends a SETUP_CLIENT message, containing:

- supported versions
- the client's role
- any extensions

The server replies on the same stream with a SETUP_SERVER message, containing:

- the selected version
- the server's role
- any extensions

The session remains active until the Session stream is closed or reset by either endpoint.
The stream MUST NOT contain any other messages.

A future version of this draft will use this session stream for other functionality, for example authentication.

## Announce Stream
A publisher can open Announce Streams to advertise a broadcast.

~~~
ANNOUNCE Message {
  Broadcast Name (b),
}
~~~

The announcement is active until the stream is closed or reset by either endpoint.
Notably, a subscriber can close the send side of the stream to indicate that no error occurred, but it's not interested.

The subscriber MUST reply with an ANNOUNCE_OK message or close the stream.

~~~
ANNOUNCE_OK Message {
  Cool = 0x1
}
~~~

## Subscribe Stream
A subscriber can open a Subscribe Stream to request a named track within a broadcast.

The subscriber MUST start the stream with a SUBSCRIBE message and MAY follow with subsequent SUBSCRIBE_UPDATE messages.

The publisher MUST reply with a SUBSCRIBE_OK, or reset the stream to reject the subscription.

The publisher then transmits any matching object within the track over a separate Object stream.
If a OBJECT is unavailable or reset, the publisher MUST reply with a OBJECT_DROP message.

The subscription is active until either endpoint closes or resets their side of the stream.
To ensure reliable delivery, an endpoint SHOULD wait until all OBJECT messages, or a corresponding OBJECT_DROP message, have been delivered before closing the stream.

### SUBSCRIBE
SUBSCRIBE is sent by a subscriber to start a subscription.

~~~
SUBSCRIBE Message {
  Subscribe ID (i)
  Broadcast Name (b)
  Track Name (b)
  Priority (i)
  Order (i)
  Min Object (i)
  Max Object (i)
}
~~~

**Priority**: The transmission priority of the subscription relative to all other active fetches and subscriptions.within the session. The publisher SHOULD transmit *lower* values first during congestion.

**Order**: The transmission order of the groups within the subscription. The publisher SHOULD transmit objects based on their sequence number in default (0), ascending (1), or descending (2) order.

**Min Group**: The minimum object sequence number plus 1. A value of 0 indicates the latest object as determined by the publisher.

**Max Group**: The maximum object sequence number plus 1. A value of 0 indicates there is no maximum.

If the subscription is accepted, the publisher will start transmitting groups for the requested track.
Object streams are transmitted according to Priority and Order on a best effort manner, but are not guaranteed to arrive in any order.


### SUBSCRIBE_OK
The publisher acknowledges the SUBSCRIBE with SUBSCRIBE_OK message.
Most of the message is informational and overlaps with INFO.

~~~
SUBSCRIBE_OK Message {
  Latest Object (i)
  Default Priority (i)
  Default Order (i)
}
~~~

**Latest Object**: The latest object as currently known by the publisher. A relay without an active subscription SHOULD forward this request upstream. The latest object MAY be outside of specified Min/Max bounds.

**Default Priority**: The default priority of this track within the broadcast. Note that this is different than SUBSCRIBE, which is scoped to a session instead. The publisher SHOULD transmit subscriptions with *lower* values first during congestion.

**Default Order**: The default order of the objects within the subscription: none (0), ascending (1), or descending (2). The publisher MUST use the subscriber's order if it was specified.


### SUBSCRIBE_UPDATE
The subscriber MAY modify a subscription with a SUBSCRIBE_UPDATE message.

~~~
SUBSCRIBE_UPDATE Message {
  Priority (i)
  Order (i)
  Min Object (i)
  Max Object (i)
}
~~~

**Min Object**: The new minimum object sequence, or 0 if there is no update. This value MUST NOT be smaller than prior SUBSCRIBE and SUBSCRIBE_UPDATE messages.

**Max Object**: The new maximum object sequence, or 0 if there is no update. This value MUST NOT be larger than prior SUBSCRIBE or SUBSCRIBE_UPDATE messages.


### SUBSCRIBE_DROP
The publisher replies with a SUBSCRIBE_DROP message when it is unable to serve an object within the requested range.

~~~
SUBSCRIBE_DROP {
  Start Oviect (i)
  Count (i)
  Error Code (i)
}
~~~

**Start Object**: The first object within the dropped range.
**Count**: The number of additional groups after the first. The End Object can be computed as Start Object + Count.
**Error Code**: An error code indicated by the application.


## Fetch Stream
A subscriber can open a bidirectional Fetch stream to receive a single Object at a specified offset.
This is primarily used to recover from an abrupt stream termination.

~~~
FETCH Message {
  Broadcast Name (b)
  Track Name (b)
  Priority (i)
  Group (i)
  Offset (i)
}
~~~

**Priority**: The priority of the group relative to all other FETCH and SUBSCRIBE requests within the session. The publisher should transmit *lower* values first during congestion.

**Offset**: The requested offset in bytes *after* the OBJECT header.

The publisher replies on the stream with the contents.
The stream may be reset with an error code by either endpoint.

DISCUSS: Is there any merit in creating a unidirectional stream instead?


## Info Stream
The subscriber may request information about a track.

The subscriber encodes a INFO_REQUEST message and the publisher replies with an INFO message.
There MUST NOT be any subsequent messages on the stream.

### INFO_REQUEST
~~~
INFO_REQUEST Message {
  Broadcast Name (b)
  Track Name (b)
}
~~~

### INFO Message
The publisher replies with an INFO message.

~~~
INFO Message {
  Latest Object (i)
  Default Priority (i)
  Default Order (i)
}
~~~

**Latest Object**: The latest object as currently known by the publisher. A relay without an active subscription SHOULD forward this request upstream.

**Default Priority**: The producer's preferred priority of tracks within the broadcast. Note that this is different than SUBSCRIBE, which is scoped to a session instead. The publisher should transmit *lower* values first during congestion.

**Default Order**: The producer's preferred order of objects within the subscription: none (0), ascending (1), or descending (2).


# Data Messages
Unidirectional streams are used for data... with the exception of FETCH for now.

|------|-----------|-------
| ID   | Type      | Role |
|-----:|:----------|-------
| 0x0  | GROUP     | Publisher |
|------|-----------|-|

## Object Stream
The publisher creates Object Streams in response to a SUBSCRIBE.

A Group stream MUST start with a GROUP message and MAY be followed by any number of FRAME messages.

Both the publisher and subscriber MAY reset the stream at any time.
The lowest priority streams should be reset first during congestion.

When a Group stream is reset, the publisher MUST send a SUBSCRIBE_DROP message on the corresponding Subscribe stream.
A future version of this draft might utilize reliable reset instead.

### GROUP Message
The GROUP message contains information about the group, as well as a reference to the subscription being served.

~~~
GROUP Message {
  Subscribe ID (i)
  Group Sequence (i)
}
~~~

### FRAME Message
The FRAME message consists of a length followed by that many bytes.

~~~
FRAME Message {
  Payload (b)
}
~~~

**Payload**: An application specific payload.
A MoqTransfork library MUST NOT inspect or modify the contents unless otherwise negotiated.


{::boilerplate bcp14-tagged}


# Security Considerations
TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
