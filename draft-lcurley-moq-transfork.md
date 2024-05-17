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

However, there are fundamental flaws with MoqTransport.
It's been years and we're still unable to align on the most critical property of the transport, literally how to utilize QUIC.
We're attempting to find middle ground with nothing concrete on either side.
In our RUSH to standardize a protocol, the QUICR solutions have led to WARP in ideals.

This fork is meant to be constructive.
An alternative vision, where we embrace the properties of QUIC instead of shunning them.
My hope is that this draft starts a conversation even if some of my ideas are duds.

Here's a list of differences ordered from most important to least:

## Object Model
MoqTransport introduces the concept of Tracks, Groups, and Objects.
While not explicitly stated, these implicitly map to media concepts:

- Track: Audio/Video Track
- Group: Video Group of Pictures
- Object: Audio/Video Frame

The most important part of any Media over QUIC protocol is right in the name.
How do you transfer Media over QUIC?

Unfortunately, the answer for MoqTransport is "it depends".
And thus the properties of the object model are dynamic.
Each track has a "delivery preferences" that signal how the object model maps to QUIC streams.
The application/relay can further drop, reorder, and prioritize objects even within a QUIC stream.

We've managed to define so many optional properties that nothing concrete is left.
This causes a migraine for everyone involved; you can't reason about what the transport provides nor can you implement a generic library.
The object model only serves to give the same name to wildly different "objects", confusing even those most versed in the draft.

MoqTransfork takes a different approach.
QUIC provides concrete properties that MoqTransfork then extends for the purpose of live media, such as the ability to subscribe and prioritize.
The object model is static, so a "group" behaves identically regardless of the application, implementation, relay, or client.

There is no implicit mapping to media; the application instead builds on top the concrete properties provided by MoqTransfork.
For example, a "group" could be a single frame, or a group of pictures, or an SVC layer, or a chat message, or whatever the application wants.
See the appendix for more examples.

## QUIC Mapping
QUIC streams provide concrete properties.
The data within a QUIC stream is delivered reliably and in order until a STREAM_FIN, or abruptly truncated with a RESET_STREAM.
Critically for MoQ, QUIC streams are independent and can be delivered out-of-order and even prioritized.

MoqTransfork builds on top of these properties.
A Group is always a single QUIC stream, so it too inherits these ordered/reliable properties.
Each subscription operates within the bounds of QUIC, only capable of starting, dropping, and prioritizing at stream boundaries.
The contents of these streams are specific to the application and opaque to any MoqTransport library or relay.

DISCUSS: Use a different name than Group to avoid confusion?

Additionally, MoqTransfork leverages the QUIC stream state machine when possible.
There are no more messages that start with `UN` or end with `_DONE`, `_ERROR`, `_RESET`, etc.
Instead, the corresponding QUIC stream is closed or reset with an error code.

### Subscriber Priority
As the original proponent of producer choosen priorities (send order), I'm ashamed to admit that I was wrong.

The MoqTransport draft contains a `priority` field for each `OBJECT`.
The idea is that a producer encodes the priority, like `newer > older` and `audio > video`, such that any relays will obey the same scheme.

Unfortunately, reality is more nuanced.
This scheme is unable to handle many common use-cases based on the viewer, including:
- different producers: when subscribed to multiple broadcasts
- different orders: VOD vs real-time
- different preferences: muted, language, quality, etc

MoqTransfork instead delegates priority to the subscriber.
This is done via a `priority` (between tracks) and `order` (within a track) field within `SUBSCRIBE`.
The publisher indicates its preference via `INFO` but the subscriber has the final say.

When deduplicating subscriptions, a relay can resolve conflicts by using the producer's preference.
That way the viewer's preference is always used for the last hop, but upstream hops might instead use the producer's preference when serving multiple viewers.


### Byte Offsets
When a connection or subscription is severed, it's desirable to resume where it left off.

MoqTransport implements this via subscriptions that can start/end at an object ID within a group.
This is an okay approach, however it can result in redownloading objects (ex. incomplete keyframes) especially when they are delivered out of order.
But critically it can cause a fragmented cache, as a relay needs to parse incoming streams and split them at object boundaries.

MoqTransfork instead utilizes FETCH for each incomplete GROUP at a byte offset.
This ensures that only the missing tail of an incomplete stream is delivered.
It also dramatically simplifies relays, allowing them to stream from a cache by byte offset instead of a parsed marker (object ID).
An advanced relay can now even forward STREAM chunks out of order, which was not always possible with Object IDs.

### Control Streams
MoqTransport control messages are fine, but have the potential for head-of-line blocking as they share a single stream.
There's also just a lot of messages to transition state machines, such as UNSUBSCRIBE, SUBSCRIBE_ERROR, and SUBSCRIBE_DONE.

MoqTransfork continues the trend of leveraging QUIC with a separate control stream per transaction.
The stream state machine is used instead of reinventing the wheel.
The afformentioned messages have been replaced with:

- UNSUBSCRIBE: subscriber STREAM_FIN
- SUBSCRIBE_ERROR: publisher RESET_STREAM
- SUBSCRIBE_DONE: publisher STREAM_FIN

Additionally, flow control will limit the maximum number of bidirectional streams per direction, removing the need for something like MAX_SUBSCRIBES.

I concede that we may need more than an error code in the future.
But until then this is super cute and simplifies the implementation.

### No Datagrams
MoqTransport allows small objects to be sent as QUIC datagrams.
This is often useful when the desired latency is below the RTT as it avoids some overhead.

The problem with QUIC datagrams is that they do a poor job catering to higher latency subscribers.
When retransmissions are possible, such as when RTT is low and bandwidth is available, the one-and-done nature of datagrams results in a worse user experience.
This is especially true for VOD use cases, where it may have made sense to use datagrams for the original broadcast, but definitely not for the replay.

The MoqTransfork alternative is to create a QUIC stream, write the data, and send a RESET_STREAM after a short delay.
This works but incurs additional overhead and is subject to flow control.
The advantage is that payloads can be larger than the MTU and may be retransmitted when RTT < delay.

A future version of this draft may support datagrams.
They would be separate entities from groups/objects as they have different properties suited only for real-time scenarios.



# Concepts

## Object Model
The MoqTransfork object model consists of:
- Broadcast
- Track
- Group
- Object
- Datagram

The terminology is borrowed from MoqTransport but critically, the properties are fixed and do not change based on flags.

### Broadcast
A collection of tracks from a single producer.

Each broadcast is identified by a unique name within the session.
A publisher may optionally ANNOUNCE a broadcast or the subscriber can determine the name out-of-band.

The MoqTransport draft refers to this as "track namespace".
I couldn't help but bikeshed since I'd eventually like broadcasts to have more properties than just a name.

### Track
A series of groups within a broadcast,

Each subscription is scoped to a single track.
A subscription starts at a group boundary and continues until either the publisher or subscriber terminates it.
The subscriber chooses the priority of each track, dictating which track arrives first during congestion.

Each track has a unique name within the broadcast.
There is currently no way to discover tracks within a broadcast; it must be negotiated out-of-band.
For example, a `catalog` track that lists all other tracks.

### Group
A series of objects within a track, served via a QUIC stream.

Just like QUIC streams, a group is delivered reliably in order with no gaps.
There is a header that outlines any stream-level properties.

Both the publisher and subscriber can cancel a GROUP stream at any point with QUIC's RESET_STREAM or STOP_SENDING respectively.
When a group is cancelled, the publisher transmits a GROUP_DROPPED message to inform the subscriber.
A subscriber may issue a FETCH to resume at a given byte offset.


### Object
A sized payload of bytes within a group.

This currently only provides framing, which is useful in some applications.


# Control Streams
Bidirectional streams are used for control messages.

Note that QUIC bidirectional streams have both a send and recv direction can be closed or reset (with an error code) independently.
This is used to indicate completion or errors respectively.

The first varint of each stream indicates the type.
Streams may only be created by the indicated role, otherwise the session MUST be closed with a ROLE_VIOLATION.
A stream type MUST NOT contain messages defined for other stream types unless otherwise specified (ex. INFO).

|------|-----------|-------
| ID   | Type      | Role |
|-----:|:----------|-------
| 0x0  | SETUP     | Client |
|------|-----------|-|
| 0x1  | ANNOUNCE  | Publisher |
|------|-----------|-|
| 0x2  | SUBSCRIBE | Subscriber |
|------|-----------|-|
| 0x3  | FETCH     | Subscriber |
|------|-----------|--|
| 0x4  | INFO      | Subscriber |
|------|-----------|--|


## SETUP Stream
Upon establishing the QUIC/WebTransport session, the client opens a SETUP stream.

The client sends a SETUP_CLIENT message, containing:
- supported versions
- the client's role
- any extensions

The server replies on the same stream with a SETUP_SERVER message, containing:
- the selected version
- the server's role
- any extensions

The session remains active until the SETUP stream is closed or reset by either endpoint.

The stream MUST NOT contain any other messages.
A future version of this draft may use the SETUP stream for other purposes, for example authentication.

## ANNOUNCE Stream
A publisher can open multiple ANNOUNCE streams to advertise a broadcast.

~~~
ANNOUNCE Message {
  Broadcast Name (b),
}
~~~

The announcement is active until the stream is closed or reset by either endpoint.
Notably, a subscriber can close the send side of the stream to indicate that no error occurred, but it's not interested.

The subscriber MUST reply with an ANNOUNCE_OK message.

~~~
ANNOUNCE_OK Message {
  Cool = 0x1
}
~~~

## SUBSCRIBE Stream
A subscriber can open a SUBSCRIBE stream to request a named track within a broadcast.

The subscriber MUST start the stream with a SUBSCRIBE message and MAY follow with subsequent SUBSCRIBE_UPDATE messages.

The publisher MUST reply with a SUBSCRIBE_OK, or reset the stream to reject the subscription.

The publisher then transmits any matching group within the track over a separate GROUP stream.
If a GROUP is unavailable or reset, the publisher MUST reply with a  SUBSCRIBE_DROP message.

The subscription is active until either endpoint closes or resets their side of the stream.
To ensure reliable delivery, an endpoint SHOULD wait until all GROUP messages, or a corresponding SUBSCRIBE_DROP message, have been delivered before closing the stream.

### SUBSCRIBE
SUBSCRIBE is sent by a subscriber to start a subscription.

~~~
SUBSCRIBE Message {
  Subscribe ID (i)
  Broadcast Name (b)
  Track Name (b)
  Priority (i)
  Order (i)
  Min Group (i)
  Max Group (i)
}
~~~

**Priority**: The preferred priority of the groups relative to all other active FETCHs and SUBSCRIBEs within the session. The publisher should transmit *lower* values first during congestion.

**Order**: The subscriber's preferred order of the groups within the subscription. The publisher should transmit groups based on their sequence number in default (0), ascending (1), or descending (2) order.

**Min Group**: The minimum group sequence number plus 1. A value of 0 indicates the latest group as determined by the publisher.

**Max Group**: The maximum group sequence number plus 1. A value of 0 indicates there is no maximum.


### SUBSCRIBE_OK
The publisher acknowledges the SUBSCRIBE with an INFO message.
This is informational, avoiding the nerd for a separate INFO_REQUEST.
See the INFO section for details.

The publisher SHOULD use the subscriber's preferred order instead of the indicated default order.
The indicated latest group MAY be outside of the Min/Max bounds.


### SUBSCRIBE_UPDATE
The subscriber MAY modify a subscription with a SUBSCRIBE_UPDATE message.

~~~
SUBSCRIBE_UPDATE Message {
  Priority (i)
  Order (i)
  Min Group (i)
  Max Group (i)
}
~~~

**Min Group**: The new minimum group sequence, or 0 if there is no update. This value MUST NOT be smaller than prior SUBSCRIBE and SUBSCRIBE_UPDATE messages.

**Max Group**: The new maximum group sequence, or 0 if there is no update. This value MUST NOT be larger than prior SUBSCRIBE or SUBSCRIBE_UPDATE messages.


### SUBSCRIBE_DROP
The publisher replies with a SUBSCRIBE_DROP message when it is unable to serve a group within the requested range.

~~~
SUBSCRIBE_DROP {
  Start Group (i)
  Count (i)
  Error Code (i)
}
~~~

**Start Group**: The first group within the dropped range.
**Count**: The number of additional groups after the first. The End Group can be computed as Start Group + Count.
**Error Code**: An error code indicated by the application.


## FETCH Stream
A subscriber can open a bidirectional FETCH stream to receive a single GROUP at a specified offset.
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

**Priority**: The preferred priority of the group relative to all other FETCH and SUBSCRIBE requests within the session. The publisher should transmit *lower* values first during congestion.

**Offset**: The requested offset in bytes *after* the GROUP header.

The publisher replies on the stream with the contents.
The stream may be reset with an error code by either endpoint.

DISCUSS: Is there any merit in creating a unidirectional stream instead?

## INFO Stream
The subscriber may request information about a track.

The subscriber encodes a INFO_REQUEST message and the publisher replies with an INFO message.
There MUST NOT be any subsequent messages on the stream.

### INFO_REQUEST Message
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
  Latest Group (i)
  Default Priority (i)
  Default Order (i)
}
~~~

**Latest Group**: The latest group as currently known by the publisher. A relay without an active subscription SHOULD forward this request upstream.

**Default Priority**: The producer's preferred priority of tracks within the broadcast. Note that this is different than SUBSCRIBE, which is scoped to a session instead. The publisher should transmit *lower* values first during congestion.

**Default Order**: The producer's preferred order of the groups within the subscription: none (0), ascending (1), or descending (2).


# Data Streams
Unidirectional streams are used for data, with the exception of FETCH because I'm still unsure.

|------|-----------|-------
| ID   | Type      | Role |
|-----:|:----------|-------
| 0x0  | GROUP     | Publisher |
|------|-----------|-|

## GROUP Stream
The publisher creates GROUP streams in response to a SUBSCRIBE.

The stream MUST start with a GROUP message and MAY be followed by any number of OBJECT messages.

The publisher MUST send a SUBSCRIBE_DROP if the stream is reset using the corresponding error code.
This includes resets triggered by the subscriber via STOP_SENDING.

### GROUP Message
The GROUP message contains information about the group, as well as a reference to the subscription being served.

~~~
GROUP Message {
  Subscribe ID (i)
  Group Sequence (i)
}
~~~

### OBJECT Message
The OBJECT message consists of a length followed by that many bytes.

~~~
OBJECT Message {
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
