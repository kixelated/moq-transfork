---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
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
  moqt: draft-ietf-moq-transport
  quic-streams: draft-kazuho-quic-quic-on-streams


--- abstract

TODO Abstract


--- middle

# Fork

 
As one of the authors of the MoqTransport ([moqt]) draft, it pains me to fork the draft.

I absolutely believe in the motivation and potential of Media over QUIC.
The layering is phenomenal and addresses many of the problems with current live media protocols.
I fully support the goals of the working group and the IETF process.

However, it's clear to me that there's fundamental issues with the MoqTransport draft.
Primerally, we don't have a consensus on *how* to send media over QUIC which is awfully ironic.
The well-intentioned compromise is support multiple modes but this has made the protocol obtuse and difficult.
It has been miserable implementing and using a generic MoqTransport library and things are only getting worse as new use-cases are discovered.

This fork is meant to be constructive.
I've spent far too much of my professional and personal time working on MoQ and want to see it through.
But there's only so much that can be repeatedly debated in Github issues.

In our RUSH to standardize a protocol, the QUICR solutions often lead to WARP in ideals.
Perhaps it's not the best idea to design an experimental protocol by committee.
I want to focus on the implementation and lead by example.


# Differences
This is a summary of the high-level differences between the MoqTransport draft ([moqt]) and this draft.

Conceptually, MoqTransfork should be considered as an extension of QUIC.
It adds properties to support live media but it does not attempt to replace any QUIC functionality.

This draft (moq-transfork-00) is based on moq-transport-03.
Unless otherwise indicated, the procedures and encoding are the same.
Future versions of moq-transfork will be independent.

# Object Model
The MoqTransfork object model consists of:
- Broadcast
- Track 
- Group
- Object
- Datagram

The terminology is borrowed from MoqTransport but critically, the properties are fixed and do not change based on flags or the application.

## Broadcast 
A collection of tracks from a single producer.

Each broadcast is identified by a unique name within the session.
A publisher may optionally ANNOUNCE a broadcast or the subscriber can determine the name out-of-band.

The MoqTransport draft refers to this as "track namespace".
I couldn't help but bikeshed.

## Track
A series of groups within a broadcast,

Each subscription is scoped to a single track.
A subscription starts at a group boundary and continues until either the publisher or subscriber terminates it.
The subscriber chooses the priority of each track, dictating which track arrives first during congestion.

Each track has a unique name within the broadcast.
There is currently no way to discover tracks within a broadcast; it must be negotiated out-of-band.
For example, a `catalog` track that lists all other tracks.

## Group
A series of objects within a track, served via a QUIC stream.

Just like QUIC streams, a group is delivered reliably in order with no gaps.
There is a header that dicates stream-level properties such as TTL.

Both the publisher and subscriber can cancel a GROUP stream at any point with QUIC's RESET_STREAM or STOP_SENDING respectively.
When a group is cancelled, the publisher transmits a GROUP_DROPPED message to inform the subscriber.
A subscriber may issue a FETCH to resume at a given byte offset.


## Object
A sized payload of bytes within a group.

This provides framing, that is useful in some applications.

## Datagram
A sized payload of bytes within a track, served via a QUIC datagram.

Datagrams members of a track that are distinct from Groups/Objects.
A publisher transmits them at most once based on the provided priority/TTL.
A datagram does not contain a sequence number and the application is responsible for any reordering.

# Messages 
## SUBSCRIBE 
~~~
SUBSCRIBE Message {
  Subscribe ID (i),
  Broadcast Name (b),
  Track Name (b),
  Priority (i),
  Order (i),
  Min Group (i),
  Max Group (i),
}
~~~



## Properties
The object model leverages QUIC streams to provide concrete properties.

The application is responsible for mapping media to the object model based on the properties it provides
The idea is the application uses the object model based on the desired properties.
This is in contrast to the MoqTransport draft, which dynamically changes the properties of the object model based on flags.

Here's a summary:

### Broadcast
- **Addressable** via a unique string.
- **Terminal** with an optional error code.

### Track
- **Addressable** via a unique string within the broadcast.
- **Terminal** with an optional error code.
- **Prioritized** via subscribe parameter.
- **Seekable** via subscribe parameters.

### Group
- **Addressable** sequence number within a track.
- **Unordered** within a track.
- **Terminal** with an optional error code.
- **Prioritized** via subscribe parameter.
- **Expires** via an optional duration.
- **Flow Controlled** via QUIC.

### Object
- **Addressable** via byte offsets within a group.
- **Ordered** within a group.
- **Sized** with a length prefix.
- **Payload** of opaque bytes.
- **Reliable** via QUIC retransmits.

## No Gaps
QUIC does not allow dropping, or prioritize, or reorder bytes in the middle of a stream.
This is by design as it simplifies the QUIC API and implementation.

MoqTransfork likewise does not support gaps.
The group/object sequence numbers are intended to be trans

If an application wants to skip an object, then it can either:
- encode a zero-length object or
- increment a timestamp within the payload

## Reliable/Ordered
Each group is delivered via a QUIC stream, providing ordering and reliablility.
As mentioned above, QUIC does not support dropping, prioriting, or reordering bytes in the middle of a stream.

If an application wants objects to be delivered/prioritized independently, then they need to be in separate groups (ex. audio) or tracks (ex. SVC).
See the Appendix for an exhaustive list of examples.



# Overview
MoqTransfork is a generic transport protocol that augments QUIC to provide extra functionality required for live media.

In particular:
- **Fanout**: A single producer can broadcast to multiple consumers.
- **Live**: All available data is transmitted without delay.
- **Prioritized**: During congestion, the most important data is transmitted first.
- **Relays**: All of the above can be accomplished with multiple hops.


## Session
MoqTransfork uses WebTransport: a thin layer on top of QUIC utilizing HTTP/3.

A client initiates a WebTransport session via a URL.
The client then initiates a MoqTransfork session, performing version/extension negotiation.
Both endpoints negotiate if they will be a publisher, subscriber, or both.

A future version of MoqTransfork will support QUIC without WebTransport.
However, until then, only WebTransport is supported to increase interoperability.

## Streams
MoqTransfork uses multiple QUIC streams to both deliver control messages and data.

## Object Model
An application builds on top of MoqTransfork by using Broadcasts, Tracks, Groups, and Objects.

### Broadcasts
A Broadcast is a named collection of tracks from a single producer.
The name is a unique identifier and relays use it route subscriptions across the network.

A publisher can choose to ANNOUNCE a broadcast, or the application can exchange the name of a broadcast out-of-band.
A subscriber can then choose to SUBSCRIBE to individual tracks within the broadcast.
The application determines any relationship between tracks, for example if they are timestamp synchronized.

There is currently no mechanism within MoqTransfork to discover track names within a broadcast.
The application is responsible for negotiating a scheme or exchanging names out-of-band.
For example, via a "catalog" track that lists all other tracks and their properties.


### Tracks
A Track is a series of objects, split into independently decodable groups.

A single publisher can serve multiple subscribers, each starting at a group boundary.
New subscribers will often start at the latest group, but they can also specify a start/end range.
A publisher may be unaware of all downstream subscribers especially when relays are involved.


### Groups
A Group is a sequence of ordered objects delivered via a QUIC stream.

Groups within a track may be delivered out-of-order and prioritized according to the subscriber's preference.
This occurs on a per-session basis and enables the receiver to skip over non-critical data during congestion.
The remainder of a group may be dropped by either endpoint if the delay is significant.

Objects within a group are delivered in-order and reliably.
This head-of-line blocking is intentional and can be used by the application to simplify a decoder.
If the objects do not depend on each other, then the application is encouraged to split them into separate groups or even tracks.

### Objects
An object is a sized payload of bytes.

The contents are opaque to MoqTransfork unless otherwise negotiated.
Objects can be used to carry media frames, metadata, control messages, or really anything.


# Implementation

## Establishment


# Control Streams
Bidirectional streams are used for control messages.
 
Note that QUIC bidirectional streams have both a send and recv direction that an endpoint can close independently. To simplify things, this document only refers to the send side of the stream as either:

- **closed**: The sender has transmitted a STREAM_FIN and all STREAM data has been acknowledged.
- **reset**: The sender has transmitted a RESET_STREAM with an error code, OR the receiver has transmitted a STOP_SENDING (which then triggers a RESET_STREAM).

The first varint of each stream indicates the type.
Streams may only be created by the indicated role, otherwise the session MUST be closed with a ROLE_VIOLATION. 

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
Upon establishing the WebTransport session, the client opens a SETUP stream.

The client sends a SETUP_CLIENT message, containing:
- supported versions
- the client's role
- any extensions

The server replies on the same stream with a SETUP_SERVER message, containing:
- the selected version
- the server's role
- any extensions

The session remains active until the SETUP stream is closed or reset by either endpoint.

A future version of this draft may use the SETUP stream for other purposes, for example authentication.

## ANNOUNCE Stream
A publisher can open multiple ANNOUNCE streams to advertise a broadcast. 

~~~
ANNOUNCE Message {
  Type = 0x1,
  Broadcast Name (b),
}
~~~

The announcement is active until the stream is closed or reset by either endpoint. Notably, a subscriber can close the send side of the stream to indicate that no error occurred but its not interested.

The subscriber MUST reply on the same stream with an OK message. 

~~~
OK Message {
  Cool = 0x1
}
~~~

## SUBSCRIBE Stream
A subscriber can open a SUBSCRIBE stream to request a track.

~~~
SUBSCRIBE Message {
  Type = 0x2,
  Subscribe ID (i),
  Broadcast Name (b),
  Track Name (b),
  Priority (i),
  Order (i),
  Min Group (i),
  Max Group (i),
}
~~~

**Priority**: The preferred priority of the groups relative to all other FETCH and SUBSCRIBE within the session. The publisher should transmit *lower* values first during congestion.

**Order**: The subscriber's preferred order of the groups within the subscription. The publisher should transmit groups based on their sequence number in any (0), ascending (1), or descending (2) order.

**Min Group**: The minimum group sequence number plus 1. A value of 0 indicates the latest group as determined by the publisher.

**Max Group**: The maximum group sequence number plus 1. A value of 0 indicates there is no maximum.

## SUBSCRIBE_OK
~~~
SUBSCRIBE_OK Message {
  Order (i)
  Min Group (i)
  Max Group (i)
}
~~~

**Order**: The publisher's preferred order of the groups within the subscription. 
This does not need to match the subscriber's preference and may be purely informational (for relays).
The publisher should transmit groups based on their sequence number in none (0), ascending (1), or descending (2) order.

**Min Group**: 

## SUBSCRIBE_UPDATE
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


## FETCH
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

# Data Streams
Unidirectional streams are used for subscription data.

|------|-----------|-------
| ID   | Type      | Role |
|-----:|:----------|-------
| 0x0  | GROUP     | Publisher |
|------|-----------|-|

## GROUP Stream
The publisher creates GROUP streams in response to a SUBSCRIBE.

~~~
GROUP Message {
  Subscribe ID (i)
  Group Sequence (i)
}
~~~

{::boilerplate bcp14-tagged}


# Security Considerations
---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
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
  moqt: draft-ietf-moq-transport
  quic-streams: draft-kazuho-quic-quic-on-streams


--- abstract

TODO Abstract


--- middle

# Fork
As one of the authors of the MoqTransport ([moqt]) draft, it pains me to fork the draft.

I absolutely believe in the motivation and potential of Media over QUIC.
The layering is phenomenal and addresses many of the problems with current live media protocols.
I fully support the goals of the working group and the IETF process.

However, it's clear that theres a problem within MoqTransport.
QUIC is still a relatively new network protocol and there are very different ideas on how to utilize it.
As a result, the draft is intentionally obtuse, has divergent modes/flags, and has few concrete properties that can be relied on.
In our RUSH to standardize a protocol, the QUICR solutions often lead to WARP in ideals.

On a personal level, I've spent far too much of my time on MoQ and I don't want to lose it all.
This fork is meant to be constructive, unlike the frequent debates about the (lack of) properties of the object model.
My goal is to lead by example and focus on the implementation.

This draft (moq-transfork-00) documents the differences from moq-transport-03.
Future versions of moq-transfork will be stand-alone.

I've summarized some of the high level changes:

## QUIC Mapping
Despite the name "Media over QUIC", the MoqTransport draft does a poor of using QUIC.
The most important part of the protocol was quite literally an afterthought. 

MoqTransfork is 

an extension of QUIC with streams and datagrams at the core.
Each stream and datagram are augmented with additional behavior 

It might seem subtle, but the 

This includes the ability to subscribe to

## Motivation
The Media over QUIC working group contains some amazing ideals and ideas.
There's been a lot of organic growth and we will eventually produce something special.
I fully support the goals of the working group and the IETF process.

However, we're in the precarious position of developing a protocol via committee using a technology still in its infacy.
With such theoritical discussions, it's easy to lose sight of the practical implications of our decisions.
And personally, it's become very draining to make forward progress.
In our RUSH to standardize a protocol, the QUICR solutions often lead to WARP in ideals.

This fork is meant to be constructive.
Sometimes the best way to argue for a solution is to present an alternative.
My hope is that some of these ideas will be adopted by the working group, even if others will be discarded.
And I'm just tired of posting hundreds of issues/messages/PRs and getting nowhere.

## Critique
A quick critique of the MoqTransport draft as why I felt the need to fork it.

### Object Model
The MoqTransport draft introduces the concept of Objects, Groups, and Tracks.
While these are generic terms and there's no explicit media mapping on purpose, the implicit mapping is:

- **Object** = Audio/Video Frame
- **Group** = Group of Pictures (video only)
- **Track** = Audio/Video Track

But how do you actually transmit a media frame over QUIC?
The unfortunate answer is: *it depends*.

For example, there's a desire to drop non-reference frames in the middle of a group.
This is an advanced use-case and not all applications will need the added complexity.
Or maybe some applications don't want to drop any content at all and just want an RTMP replacement.

As a result, there's (currently) four different "delivery preferences" that an application can objects via QUIC:
- QUIC stream per Track
- QUIC stream per Group
- QUIC stream per Object
- QUIC datagram per Object

The reason is because QUIC does not allow dropping or reordering bytes in the middle of a stream *by design*.
In order to drop an object in the middle of a group, we now need to create own fragmentation mechanisms.
And of course now we need our own way of signalling FINs, RSTs, gaps, reordering, etc.

The object model creates an alarming depature from QUIC and a burden to implement.

### Properties
As mentioned above, the object model intends to map to media concepts, while being generic of course.
However, the actual properties of Objects, Groups, and Tracks are wildly variable.

For example, let's consider the properties of a Group:

- **Unique ID**: Yes
- **Sequential ID**: Depends on the application.
- **Ordered between Groups?**: Yes with a stream per track, no otherwise.
- **Ordered within a Group?**: Yes with a stream per track/group, no for a stream/datagram per object.
- **Gaps between Groups?**: Depends on the application.
- **Gaps within a Group?**: Depends on the application.
- **Drops between Groups?**: No with a stream per track, yes otherwise.
- **Drops within a Group?**: No with a stream per track, tail only with a stream per group, yes otherwise.

While these seem like concrete albeit random properties, this is just my interpretation of the draft.
The reality is that these properties are undefined and the even authors of the draft have different interpretations.
We use the same terms but mean different wildly things.

The object model puts an asterisk on every property exposed to the application developer.

### Usage of QUIC
Ironically "Media over QUIC" doesn't really leverage QUIC.

Within the working group, there's been a debate of "implicit" vs "explicit".
The names are confusing, but basically the question, is does MoqTransport use any signals from QUIC?

- Implicit: Yes, each message has an associated QUIC stream and can leverage any ordering/FIN/RESET/etc.
- Explicit: No, we can't assume QUIC is used need our own cooresponding message.

The draft has erred on the side of "explicit", which is why we have our own *_DONE, *_ERROR, *_RESET, etc signals.
But now the library has to implement it's own (undefined) state machine for each announce/subscribe operation.
An application using a stream/datagram per object has to futher implement reassembly, reordering, gap handling, fin signalling, etc.

I want to use QUIC.
I don't want to reimplement QUIC.

And if you want to use Media over QUIC over TCP, then use something like the QUIC Streams proposal [draft-kazuho-quic-quic-on-streams].


## Differences
This is a summary of the high-level differences between the MoqTransport draft ([moqt]) and this draft.

Conceptually, MoqTransfork should be considered as an extension of QUIC.
It adds properties to enable relay fanout but it does not attempt to replace any QUIC functionality.

## Object Model
The MoqTransfork object model inherits the QUIC object model rather than fighting against it.

The same terms are used for consistency: Object, Group, Track.
The one addition is Broadcast, which was implied in the MoqTransport draft as "namespace".

- **Broadcast**: A collection of tracks from a single producer, identified by a unique name.
- **Track**: A series of groups, identified by a unique name.
- **Group**: A series of objects, served via a QUIC stream.
- **Object**: A sized payload of bytes.

These are not intended to map directly to media concepts.
Depending on the application, an Object might consist of multiple frames (ex. moof) or a parital frame (ex. slice).
Likewise a Group might consist of non-dependent (ex. audio, multiple GoPs) or a layer within a GoP (ex. SVC).
See the Appendix for an exhaustive list of use-cases and examples.

## Properties
The object model leverages QUIC streams to provide concrete properties.

The idea is the application uses the object model based on the desired properties.
This is in contrast to the MoqTransport draft, which dynamically changes the properties of the object model based on flags.

Here's a summary:

### Broadcast
- **Addressable** via a unique string.
- **Terminal** with an optional error code.

### Track
- **Addressable** via a unique string within the broadcast.
- **Terminal** with an optional error code.
- **Prioritized** via subscribe parameter.
- **Seekable** via subscribe parameters.

### Group
- **Addressable** sequence number within a track.
- **Unordered** within a track.
- **Terminal** with an optional error code.
- **Prioritized** via subscribe parameter.
- **Expires** via an optional duration.
- **Flow Controlled** via QUIC.

### Object
- **Addressable** via byte offsets within a group.
- **Ordered** within a group.
- **Sized** with a length prefix.
- **Payload** of opaque bytes.
- **Reliable** via QUIC retransmits.

## No Gaps
QUIC does not allow dropping, or prioritize, or reorder bytes in the middle of a stream.
This is by design as it simplifies the QUIC API and implementation.

MoqTransfork likewise does not support gaps.
The group/object sequence numbers are intended to be trans

If an application wants to skip an object, then it can either:
- encode a zero-length object or
- increment a timestamp within the payload

## Reliable/Ordered
Each group is delivered via a QUIC stream, providing ordering and reliablility.
As mentioned above, QUIC does not support dropping, prioriting, or reordering bytes in the middle of a stream.

If an application wants objects to be delivered/prioritized independently, then they need to be in separate groups (ex. audio) or tracks (ex. SVC).
See the Appendix for an exhaustive list of examples.



# Overview
MoqTransfork is a generic transport protocol that augments QUIC to provide extra functionality required for live media.

In particular:
- **Fanout**: A single producer can broadcast to multiple consumers.
- **Live**: All available data is transmitted without delay.
- **Prioritized**: During congestion, the most important data is transmitted first.
- **Relays**: All of the above can be accomplished with multiple hops.


## Session
MoqTransfork uses WebTransport: a thin layer on top of QUIC utilizing HTTP/3.

A client initiates a WebTransport session via a URL.
The client then initiates a MoqTransfork session, performing version/extension negotiation.
Both endpoints negotiate if they will be a publisher, subscriber, or both.

A future version of MoqTransfork will support QUIC without WebTransport.
However, until then, only WebTransport is supported to increase interoperability.

## Streams
MoqTransfork uses multiple QUIC streams to both deliver control messages and data.

## Object Model
An application builds on top of MoqTransfork by using Broadcasts, Tracks, Groups, and Objects.

### Broadcasts
A Broadcast is a named collection of tracks from a single producer.
The name is a unique identifier and relays use it route subscriptions across the network.

A publisher can choose to ANNOUNCE a broadcast, or the application can exchange the name of a broadcast out-of-band.
A subscriber can then choose to SUBSCRIBE to individual tracks within the broadcast.
The application determines any relationship between tracks, for example if they are timestamp synchronized.

There is currently no mechanism within MoqTransfork to discover track names within a broadcast.
The application is responsible for negotiating a scheme or exchanging names out-of-band.
For example, via a "catalog" track that lists all other tracks and their properties.


### Tracks
A Track is a series of objects, split into independently decodable groups.

A single publisher can serve multiple subscribers, each starting at a group boundary.
New subscribers will often start at the latest group, but they can also specify a start/end range.
A publisher may be unaware of all downstream subscribers especially when relays are involved.


### Groups
A Group is a sequence of ordered objects delivered via a QUIC stream.

Groups within a track may be delivered out-of-order and prioritized according to the subscriber's preference.
This occurs on a per-session basis and enables the receiver to skip over non-critical data during congestion.
The remainder of a group may be dropped by either endpoint if the delay is significant.

Objects within a group are delivered in-order and reliably.
This head-of-line blocking is intentional and can be used by the application to simplify a decoder.
If the objects do not depend on each other, then the application is encouraged to split them into separate groups or even tracks.

### Objects
An object is a sized payload of bytes.

The contents are opaque to MoqTransfork unless otherwise negotiated.
Objects can be used to carry media frames, metadata, control messages, or really anything.


# Implementation

## Establishment


# Control Streams
Bidirectional streams are used for control messages.
 
Note that QUIC bidirectional streams have both a send and recv direction that an endpoint can close independently. To simplify things, this document only refers to the send side of the stream as either:

- **closed**: The sender has transmitted a STREAM_FIN and all STREAM data has been acknowledged.
- **reset**: The sender has transmitted a RESET_STREAM with an error code, OR the receiver has transmitted a STOP_SENDING (which then triggers a RESET_STREAM).

The first varint of each stream indicates the type.
Streams may only be created by the indicated role, otherwise the session MUST be closed with a ROLE_VIOLATION. 

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
Upon establishing the WebTransport session, the client opens a SETUP stream.

The client sends a SETUP_CLIENT message, containing:
- supported versions
- the client's role
- any extensions

The server replies on the same stream with a SETUP_SERVER message, containing:
- the selected version
- the server's role
- any extensions

The session remains active until the SETUP stream is closed or reset by either endpoint.

A future version of this draft may use the SETUP stream for other purposes, for example authentication.

## ANNOUNCE Stream
A publisher can open multiple ANNOUNCE streams to advertise a broadcast. 

~~~
ANNOUNCE Message {
  Type = 0x1,
  Broadcast Name (b),
}
~~~

The announcement is active until the stream is closed or reset by either endpoint. Notably, a subscriber can close the send side of the stream to indicate that no error occurred but its not interested.

The subscriber MUST reply on the same stream with an OK message. 

~~~
OK Message {
  Cool = 0x1
}
~~~

## SUBSCRIBE Stream
A subscriber can open a SUBSCRIBE stream to request a track.

~~~
SUBSCRIBE Message {
  Type = 0x2,
  Subscribe ID (i),
  Broadcast Name (b),
  Track Name (b),
  Priority (i),
  Order (i),
  Min Group (i),
  Max Group (i),
}
~~~

**Priority**: The preferred priority of the groups relative to all other FETCH and SUBSCRIBE within the session. The publisher should transmit *lower* values first during congestion.

**Order**: The preferred order of the groups within the subscription. The publisher should transmit groups based on their sequence number in any (0), ascending (1), or descending (2) order.

**Min Group**: The minimum group sequence number plus 1. A value of 0 indicates the latest group as determined by the publisher.

**Max Group**: The maximum group sequence number plus 1. A value of 0 indicates there is no maximum.

The subscription remains active until closed or reset by either endpoint. The subscriber SHOULD close after all groups within the bounds have been fully received or dropped. The publisher MAY close after all groups have been acknowledged or dropped if supported by the QUIC library.

The publisher replies with a INFO message containing the minimum group ID

The subscriber may UPDATE the subscription by encoding subsequent messages on the stream:

~~~
UPDATE Message {
  Priority (i)
  Order (i)
  Min Group (i)
  Max Group (i)
}
~~~

**Min Group**: The new minimum group sequence, or 0 if there is no update. This value MUST NOT be smaller than prior SUBSCRIBE and UPDATE messages.

**Max Group**: The new maximum group sequence, or 0 if there is no update. This value MUST NOT be larger than prior SUBSCRIBE or UPDATE messages.


## FETCH Stream
A subscriber can open a FETCH stream to receive a single GROUP at a specified offset.
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

**Offset**: The requested offset in bytes *after* the GROUP header, but including and OBJECT headers.

The publisher replies with the contents on the same stream. See the GROUP section on how to parse the contents which includes OBJECT delimiters.

The publisher closes the stream once the entire GROUP has been transferred, or resets with an error code.

The subscriber may close or reset the 


DISCUSS: This is notably different from SUBSCRIBE. Is there any merit in creating a unidirectional stream instead?

## INFO Stream

# Data Streams
Unidirectional streams are used for subscription data.

|------|-----------|-------
| ID   | Type      | Role |
|-----:|:----------|-------
| 0x0  | GROUP     | Publisher |
|------|-----------|-|

## GROUP Stream
The publisher creates GROUP streams in response to a SUBSCRIBE.

~~~
GROUP Message {
  Subscribe ID (i)
  Group Sequence (i)
}
~~~

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
