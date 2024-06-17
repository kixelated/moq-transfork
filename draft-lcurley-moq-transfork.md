---
title: "Media over QUIC - Transfork"
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
  moqt: I-D.ietf-moq-transport

informative:

--- abstract

MoqTransfork is a live media transport utilizing QUIC.
The protocol is designed to scale and simultaneously serve viewers with different latency/quality targets: the entire spectrum between real-time and VOD.
MoqTransfork is media agnostic as the specific media encoding (ex. container+codec) is delegated to a higher layer.

--- middle

# Conventions and Definitions
{::boilerplate bcp14-tagged}


# Fork
This draft is based on moq-transport-03 [moqt].
The concepts, motivations, and terminology are very similar on purpose.
When in doubt, refer to the original draft.

I absolutely believe in the motivation and potential of Media over QUIC.
The layering is phenomenal and addresses many of the problems with current live media protocols.
I fully support the goals of the working group and the IETF process.

However, there are practical and conceptual flaws with MoqTransport that need to be addressed.
It's been very difficult to design an experimental protocol via committee and over two years in, we're still unable to align on the most critical property of the transport... how to utilize QUIC.
The MoqTransport draft contains multiple mechanisms ("delivery preference") that are difficult to implement, use, and even discuss.
In our RUSH to standardize a protocol, the QUICR solutions have led to WARP in ideals.

This fork is meant to be constructive; an alternative vision.
I'd like to try leading by example, demonstrating that it's possible to simplify the protocol and still support a documented set of use-cases.
A fork lets me focus on building applications and gives the standard a chance to catch up instead of lead.

The appendix contains a list of high level differences between MoqTransport and MoqTransfork.

Here's an overview of the notable differences between MoqTransport and MoqTransfork:

## Object Model
The object model is an abstraction exposed to the application, forming the basis of the transport API.

The MoqTransport object model vaguely maps to media concepts, where a Group is a Video GoP and an Object is a Video Frame.
However, there's no agreed upon way to utilize QUIC, leading to the compromise that the publisher chooses a "delivery preference" for each track.
As a the result the properties of the object model change dynamically at runtime, as the properties of a QUIC stream (or datagram) are moved between a track, group, or object.
The number of permutations quickly becomes unmanageable and every conversion has to be caveated with "when using a stream per X".

The MoqTransfork object model is instead static and maps directly to QUIC.
A Group is always an ordered set of bytes, served via a QUIC stream or datagram depending on the subscription.
This simplification is able to support all of the documented use-cases; see the Appendix.

## Prioritization
Prioritization is important for low-latency, ensuring the publisher sends the most important media first during congestion.

MoqTransport uses producer chosen priorities via send order.
As the original proponent of this approach, I'm ashamed to admit that I was wrong.
Reality is more nuanced; both the subscriber and publisher need to work together.

MoqTransfork instead delegates the priority decision for the last mile to the subscriber.
This is done via a `Track Priority` and `Group Order` field within `SUBSCRIBE`.
If there's a single viewer, then this priority can be used all the way to origin.
However when there are multiple conflicting viewers, then a relay should use the producer's preference as advertised in `INFO`.

## Control Streams
At the core of any transport are control messages.

MoqTransport control messages are fine, but have the potential for head-of-line blocking as they share a single stream.
There's also just a lot of messages for state transitions such as `UNSUBSCRIBE`, `SUBSCRIBE_ERROR`, `SUBSCRIBE_DONE`, etc.

MoqTransfork continues the trend of leveraging QUIC with separate control streams, like one stream per subscription.
These control streams are terminated when endpoints close or reset (including an error code) the stream in order to directly leverage QUIC's stream state machine.
This removes any messages that start with `UN` or end with `_DONE`, `_ERROR`, `_RESET`, etc.

## Byte Offsets
When a connection or subscription is severed, it's often desirable to resume where it left off.

MoqTransport implements this via subscriptions that can start/end at an object ID within a group.
This is an okay approach, however it results in redownloading partial objects and results in a fragmented cache at parsed boundaries.

MoqTransfork instead utilizes FETCH to serve incomplete Groups starting at a byte offset.
QUIC streams are tail dropped when reset so there's no need for a more complex mechanism.
This means a relay doesn't need to parse frame/object boundaries and it can even forward STREAM frames out-of-order.

## Datagrams
Datagrams are useful when payloads are small, overhead is important, and the desired latency is below the RTT.

MoqTransport supports sending objects as QUIC datagrams via a publisher track preference.
This can work for real-time viewers but results in a poor experience for higher latency viewers subscribed to the same track.

MoqTransfork instead lets the subscriber indicate if it wants to receive a subscription via streams or datagrams.
Datagrams should only be used when Groups are smaller than the MTU and the desired latency is smaller than the RTT.
In doing so, Groups sent via datagrams may be silently dropped for whatever reason, saving a few bytes on the wire (~10 per Group) which is desirable for audio use-cases.

## Use-Cases
It's boring, but you should write down what you're trying to accomplish before you start designing a protocol especially by committee.

MoqTransport is intentionally vague and doesn't mention any use-cases.
This has caused unnecessary or incomplete features, as it's impossible to argue against a feature or suggest alternatives when "some application might need it".

MoqTransfork instead includes an Appendix contains a number of media use-cases and recommended approaches that are by no means required or comprehensive.
This also serves to illustrate the careful layering of Media over QUIC which has otherwise not been documented.



# Concepts
Many of the concepts are borrowed from MoqTransport so I'm not going to fully rehash them here; a future draft will.

## Object Model
The MoqTransfork object model consists of:

- **Broadcast**: A collection of Tracks from a single producer.
- **Track**: A series of Groups within a Broadcast.
- **Group**: A series of Frames within a Track, served in order.
- **Frame**: A sized payload of bytes within a Group.

### Broadcast
A Broadcast is a collection of tracks from a single producer, identified by a unique name within the session.

Each subscription is scoped to a single Broadcast and Track within it.
A publisher may advertise available broadcasts via an ANNOUNCE message or an out-of-band mechanism.

The MoqTransport draft refers to this as "track namespace".
I couldn't help but bikeshed.

### Track
A Track is a series of Groups within a Broadcast, identified by a unique name within the Broadcast.

Each subscription is scoped to a single Track and starts/ends at a Group boundary.
A subscriber chooses the priority of each subscription, dictating which Track arrives first during congestion.

There is currently no way to discover tracks within a broadcast; it must be negotiated out-of-band.

### Group
A Group is an ordered stream of Frames within a Track.

A Group may be served via a QUIC stream or datagram, depending on the subscription.
If the Group is dropped for any reason, a subscriber may FETCH it again starting at a given byte offset.

### Frame
A Frame is a payload of bytes within a Group.

Frames currently only provides framing, hence the name.
Framing is useful in some applications but can also be redundant or increase memory usage, as the size must be known upfront.

Frame will be removed in a future version of the draft and delegated to a higher layer.
However, it's useful when being compared to an Object in MoqTransport.

## Streams vs Datagrams
MoqTransfork supports two primary methods of transmitting data: QUIC streams and datagrams.
The difference between the two is subtle but important.

Delivering a Group via a stream ensures the Group is fully delivered or explicitly dropped.
QUIC streams do not have an upfront size, allowing the Group to be transmitted in Frame chunks over time until a STREAM_FIN marker.
This is the preferred mechanism for delivering Groups as the QUIC library will automatically provide fragmentation, retransmissions, and flow control.

Delivering a Group via a datagram instead means a Group will be transmitted once and can be silently dropped for any reason.
Each Group MUST have an upfront size that is smaller than network MTU, limiting them in both size and duration.
These restrictions allow the Group to be delivered with less overhead than a stream (~10 bytes per group) which is significant for some real-time use-cases.

A subscriber is responsible for choosing if a subscription is served via streams or datagrams.

# Workflow
This section outlines the flow of messages within a MoqTransfork session.
See the section for Messages section for the specific encoding.

## Connection
MoqTransfork runs on top of either WebTransport or QUIC directly.

WebTransport is a layer on top of QUIC and HTTP/3 required for web support.
The API is nearly identical to QUIC, however notably lacks stream IDs and has fewer available error codes.

QUIC may be used directly with the ALPN `moqf-00`, which will be updated any time there is a breaking change to version negotiation.
The QUIC client SHOULD include a PATH and ORIGIN parameter in the SESSION_CLIENT message to emulate a WebTransport client.

## Termination
MoqTransfork involves multiple concurrent streams intended to be closed or reset (with an error code) independently.
This is more involved because QUIC bidirectional streams have both a send and receive direction.

To terminate a stream, an endpoint closes (STREAM_FIN) or resets (RESET_STREAM) the send direction.
Any messages/data on a closed stream will still be delivered, while a reset results in immediate termination.
An endpoint MAY reset the receive direction (STOP_SENDING).

An endpoint MUST monitor the read direction to detect if it is closed or reset and unless otherwise specified, MUST correspondingly close or reset the send direction.
An endpoint MAY monitor the send direction for errors, especially for unidirectional streams.

An endpoint MUST close the connection on a fatal error, such as a protocol or encoding violation.


## Handshake
After a connection is established, the client opens the Session Stream and sends a SESSION_CLIENT message and the server replies with a SESSION_SERVER message.
The session is active until either endpoint closes or resets the Session Stream.

This session handshake is used to negotiate the MoqTransfork version, role, and any future parameters.


## Bidirectional Streams
Bidirectional streams are primarily used for control streams.

The first byte of each stream indicates the Stream Type.
Streams may only be created by the indicated role, otherwise the session MUST be closed with a ROLE_VIOLATION.

|------|-----------|------------|
| Byte | Type      | Role       |
|-----:|:----------|------------|
| 0x0  | Session   | Client     |
|------|-----------|------------|
| 0x1  | Announce  | Publisher  |
|------|-----------|------------|
| 0x2  | Subscribe (Streams)    | Subscriber |
|------|------------------------|------------|
| 0x3  | Subscribe (Datagrams)  | Subscriber |
|------|-----------|------------|
| 0x4  | Fetch     | Subscriber |
|------|-----------|------------|
| 0x5  | Info      | Subscriber |
|------|-----------|------------|


### Session
The Session stream contains all messages that are session level.

The client MUST open a single Session Stream immediately after establishing the QUIC/WebTransport session.
The client sends a SESSION_CLIENT message and the server replies with a SESSION_SERVER message.
Afterwards, both endpoints SHOULD send SESSION_UPDATE messages, such as after a significant change in the session bitrate.

The session remains active until the Session Stream is closed or reset by either endpoint.


### Announce
A publisher can open an Announce Stream to advertise a broadcast.
This is optional, as the application determine the broadcast name out-of-band.

The publisher MUST start the stream with an ANNOUNCE message.
The subscriber MUST reply with an ANNOUNCE_OK message or reset the stream.
The announcement is active until the stream is closed or reset by either endpoint.

There is currently no expectation that a relay will forward an ANNOUNCE message downstream.
A future draft may introduce a mechanism to discover broadcasts matching a prefix.

### Subscribe
A subscriber can open a Subscribe Stream to request a named track within a broadcast.
The Stream Type indicates if the subscription is served via QUIC streams or datagrams.

The SUBSCRIBE message contains a requested Broadcast and Track.
It also contains prioritization information and a range of Groups, all of which MAY be updated by the subscriber via a SUBSCRIBE_UPDATE message.
The subscription is active until the either endpoint closes or resets the stream.

The subscriber MUST start a Info Stream with a SUBSCRIBE message followed by any number SUBSCRIBE_UPDATE messages.
The publisher MUST reply with an INFO message followed by any number of GROUP_DROPPED messages (streams only).
A publisher MAY reset the stream at any point if it is unable to serve the subscription.

**When QUIC streams are used**, the publisher MUST transmit a complete Group Stream or a GROUP_DROPPED message for each Group within the subscription range.
This means the publisher MUST transmit a GROUP_DROPPED if a Group Stream is reset.
The subscriber SHOULD close the subscription when all GROUP and GROUP_DROP messages have been received, and the publisher MAY close the subscription after all messages have been acknowledged.

**When QUIC datagrams are used**, the publisher MAY transmit a Group/Frame Datagram for each Group within the subscription range.
The publisher MUST NOT transmit a GROUP_DROPPED message.
The publisher SHOULD close the subscription after all Group/Frame Datagrams have been transmitted, and the subscriber MAY close the subscription after all Groups have been received.


### Fetch
A subscriber can open a Fetch Stream to receive a single Group at a specified offset.
This is primarily used to recover from an abrupt stream termination, causing the truncation of a Group.

The subscriber MUST start a Fetch Stream with a FETCH message followed by any number of FETCH_UPDATE messages.
The publisher MUST reply with the contents of a Group Stream, except starting at the specified offset *after* the GROUP message.
Note that this includes any FRAME messages.

The fetch is active until both endpoints close the stream, or either endpoint resets the stream.


### Info
A subscriber can open an Info Stream to request information about a track.
This is not often necessary as SUBSCRIBE will trigger an INFO reply.

The subscriber MUST start the stream with a INFO_REQUEST message.
The publisher MUST reply with an INFO message or reset the stream.
Both endpoints MUST close the stream afterwards.


## Unidirectional
Unidirectional streams are used for subscription data.

|------|--------|-----------|
| ID   | Stream | Role      |
|-----:|:-------|-----------|
| 0x0  | Group  | Publisher |
|------|--------|-----------|

### Group
A publisher creates Group Streams in response to a Subscribe Stream.

A Group Stream MUST start with a GROUP message and MAY be followed by any number of FRAME messages.
An application MAY use zero length Group or Frames to signal gaps.

Both the publisher and subscriber MAY reset the stream at any time.
When a Group stream is reset, the publisher MUST send a GROUP_DROP message on the corresponding Subscribe stream.
A future version of this draft may utilize reliable reset instead.


## Datagrams
Datagrams are used for subscription data.

|------|--------|-----------|
| ID   | Type   | Role      |
|-----:|:-------|-----------|
| 0x0  | Group  | Publisher |
|------|--------|-----------|
| 0x1  | Frame  | Publisher |
|------|--------|-----------|

### Group Datagram
A Group Datagram consists of a GROUP message followed by one or more FRAME messages.
These contents are identical to a Group Stream.

### Frame Datagram
A Frame Datagram consists of a GROUP message followed by the Frame Payload.
This saves 1-2 bytes by skipping the FRAME header when there's a only single Frame, as the datagram itself contains a length.


# Encoding
This section covers the encoding of each message.

Note that none of these message have a type identifier.
The message is determined by the stream type and the current state.

## SESSION_CLIENT
TODO copy from moq-transport.

This contains:

- supported versions
- the client's role
- any extensions

## SESSION_SERVER
TODO copy from moq-transport

This contains:

- the selected version
- the server's role
- any extensions

## SESSION_UPDATE

~~~
SESSION_UPDATE Message {
  Session Bitrate (i)
}
~~~

**Session Bitrate**:
The estimated bitrate of the QUIC connection in bits per second.
This SHOULD be sourced directly from the QUIC congestion controller.
A value of 0 indicates that this information is not available.


## ANNOUNCE
A publisher sends an ANNOUNCE message to advertise a broadcast.

~~~
ANNOUNCE Message {
  Broadcast Name (b),
}
~~~

## ANNOUNCE_OK
A subscriber replies to an ANNOUNCE with an ANNOUNCE_OK message.

~~~
ANNOUNCE_OK Message {
  Cool = 0x1
}
~~~

## SUBSCRIBE
SUBSCRIBE is sent by a subscriber to start a subscription.

~~~
SUBSCRIBE Message {
  Subscribe ID (i)
  Broadcast Name (b)
  Track Name (b)
  Track Priority (i)
  Group Order (i)
  Group Expires (i)
  Group Min (i)
  Group Max (i)
}
~~~

**Track Priority**:
The transmission priority of the subscription relative to all other active subscriptions within the session.
The publisher SHOULD transmit *lower* values first during congestion.

**Group Order**:
The transmission order of the Groups within the subscription.
The publisher SHOULD transmit groups based on their sequence number in default (0), ascending (1), or descending (2) order.

**Group Expires**:
A duration in milliseconds that applies to all Groups within the subscription.
The group SHOULD be dropped if this duration has elapsed after group has finished, including any time spent cached.
The publisher's Group Expires value (via INFO) SHOULD be used instead when smaller.

**Group Min**:
The minimum group sequence number plus 1.
A value of 0 indicates the latest Group Sequence as determined by the publisher.

**Group Max**:
The maximum group sequence number plus 1.
A value of 0 indicates there is no maximum and the subscription continues indefinitely.


## SUBSCRIBE_UPDATE
A subscriber can modify a subscription with a SUBSCRIBE_UPDATE message.

~~~
SUBSCRIBE_UPDATE Message {
  Track Priority (i)
  Group Order (i)
  Group Expires (i)
  Group Min (i)
  Group Max (i)
}
~~~

**Group Min**:
The new minimum group sequence, or 0 if there is no update.
This value MUST NOT be smaller than prior SUBSCRIBE and SUBSCRIBE_UPDATE messages.

**Group Max**:
The new maximum group sequence, or 0 if there is no update.
This value MUST NOT be larger than prior SUBSCRIBE or SUBSCRIBE_UPDATE messages.


## INFO
The INFO message contains the current information about a track.

~~~
INFO Message {
  Track Priority (i)
  Group Latest (i)
  Group Order (i)
  Group Expires (i)
}
~~~

**Track Priority**:
The priority of this track within the broadcast.
Note that this is slightly different than SUBSCRIBE, which is scoped to a session not broadcast.
The publisher SHOULD transmit subscriptions with *lower* values first during congestion.


**Group Latest**:
The latest group as currently known by the publisher.
A relay without an active subscription SHOULD forward this request upstream

**Group Order**:
The publisher's intended order of the groups within the subscription: none (0), ascending (1), or descending (2).

**Group Expires**:
A duration in milliseconds.
The group SHOULD be dropped if this duration has elapsed after group has finished, including any time spent cached.
The Subscriber's Group Expires value SHOULD be used instead when smaller.


## INFO_REQUEST
The INFO_REQUEST message is used to request an INFO response.

~~~
INFO_REQUEST Message {
  Broadcast Name (b)
  Track Name (b)
}
~~~


## FETCH
A subscriber can request a byte offset within a Group with a FETCH message.

~~~
FETCH Message {
  Broadcast Name (b)
  Track Name (b)
  Track Priority (i)
  Group Sequence (i)
  Group Offset (i)
}
~~~

**Track Priority**:
The priority of the group relative to all other FETCH and SUBSCRIBE requests within the session.
The publisher should transmit *lower* values first during congestion.

**Group Offset**:
The requested offset in bytes *after* the GROUP message.


## FETCH_UPDATE
A subscriber can modify a FETCH request with a FETCH_UPDATE message.

~~~
FETCH_UPDATE Message {
  Track Priority (i)
}
~~~

**Track Priority**:
The priority of the group relative to all other FETCH and SUBSCRIBE requests within the session.
The publisher should transmit *lower* values first during congestion.


## GROUP
The GROUP message contains information about a Group, as well as a reference to the subscription being served.

~~~
GROUP Message {
  Subscribe ID (i)
  Group Sequence (i)
}
~~~

## GROUP_DROP
A publisher transmits a GROUP_DROP message when it is unable to serve a group for a SUBSCRIBE.

~~~
GROUP_DROP {
  Group Start (i)
  Group Count (i)
  Group Error Code (i)
}
~~~


**Group Start**:
The sequence number for the first group within the dropped range.

**Group Count**:
The number of additional groups after the first.
This value is 0 when only one group is dropped.

**Error Code**:
An error code indicated by the application.


## FRAME
The FRAME message consists of a length followed by that many bytes.

~~~
FRAME Message {
  Payload (b)
}
~~~

**Payload**:
An application specific payload.
A generic library or relay MUST NOT inspect or modify the contents unless otherwise negotiated.

# Appendix: Changelog
Notable changes between versions of this draft.

## moq-transfork-01
- Moved Expires from GROUP to SUBSCRIBE
- Added FETCH_UPDATE

## moq-transfork-00
Based on moq-transport-03

### Bikeshedding
Couldn't help it.

- Renamed Track Namespace to Broadcast
- Renamed Object to Frame


### Subscriber's Choice
MoqTransfork moves most decision making to the subscriber, so a single publisher can support multiple diverse subscribers.
The publisher provides a default value to resolve conflicts when deduplicating.

- Removed Delivery Preference (publisher choice)
-- Stream per Track
-- Stream per Group
-- Stream per Object
-- Datagram per Object
- Added Subscribe Mode (subscriber choice)
-- Stream per Group
-- Datagram per Group
- Removed Stream priority (publisher choice)
- Added Subscribe Track Priority (subscriber choice)
- Added Subscribe Group Order (subscriber choice)
- Added Group Expires (subscriber and publisher choice)

### Control Streams
Transactions like Announce and Subscribe use their own control stream, inheriting the stream state machine for error handling.

- Removed ANNOUNCE_ERROR
- Removed ANNOUNCE_DONE
- Removed UNANNOUNCE
- Removed SUBSCRIBE_OK
- Removed SUBSCRIBE_ERROR
- Removed SUBSCRIBE_DONE
- Removed UNSUBSCRIBE

### Unambiguous Delivery
The subscriber now knows if a group/frame will be delivered or was dropped.

- Group Sequences are sequential
- All sequences within the SUBSCRIBE range are delivered or dropped.
- GROUP_DROP when a group is dropped.

### Track INFO
Added a mechanism to request information about the current track state.

- Added INFO_REQUEST and INFO
- Replaced SUBSCRIBE_OK with INFO

### Fetch via Offset
A reconnecting subscriber can request the retransmission of a group/stream at a given byte offset.

- Added FETCH.
- Removed Object ID.


# Appendix: Media Use-Cases
These are some recommended ways to use MoqTransfork for media delivery.

## Video
Video encoding involves complex dependencies between frames/slices.
The terminology in this section stems from H.264 but is applicable to most modern codecs.

Each frame of video is encoded as one or more slices but to simplify the discussion, we'll refer to a slice as a frame.
There are three types of frames:

- **I-Frame**: A frame that can be decoded independently.
- **P-Frame**: A frame that depends on previous frames.
- **B-Frame**: A frame that depends on previous or future frames.

### Group of Pictures
A simple application can ignore the complexity of P/B frames and focus on I-Frames.
This is the optimal approach for many encoding configurations.

Each I-Frame begins a Group of Pictures (GoP).
A GoP is a set of frames that MAY depend on each other and MUST NOT depend on other GoPs.
Each frame has a decode order (DTS) and a frame MUST NOT depend on frames with a higher DTS.

This perfectly maps to a QUIC stream, as they too are independent and ordered.
The easiest way to use MoqTransfork is to send each GoP as a GROUP with each frame as a FRAME, hence the names.

Given the nature of QUIC streams and GoPs, all actions are done at Group boundaries.
Each SUBSCRIBE starts at a Group to ensure that it starts with an I-Frame.
A FETCH can be used to start at a byte offset instead, assuming some of the Group has already been received.
Each Group is delivered in decode order ensuring that all frames are decodable (no artifacts).

A subscriber can choose the Group Order based on the desired user experience:

- `SUBSCRIBE order=DESC`: Transmits new Groups first to allow skipping, intended for low-latency live streams.
- `SUBSCRIBE order=ASC`: Transmits old Groups first to avoid skipping, intended for VOD and reliable live streams.

A publisher or subscriber can skip the remainder of a Group by resetting a Group Stream or by issuing a SUBSCRIBE_UPDATE.
This truncates the Group and a subscriber can issue a FETCH if it wants to resume at a byte offset.

### Layers
An advanced application can subdivide a GoP into layers.

The most comprehensive way to do this is with Scalable Video Coding (SVC).
There is a base layer and one or more enhancement layers that depend on lower layers.
For example, a 4K stream could be broken into 4K, 1080p, and 360p (base) layers.
However, SVC has limited support and is complex to encode.

Another approach is to use temporal scalability via something like B-pyramids.
The frames within a GoP are sub-divided based on their dependencies, intentionally creating a hierarchy.
For example, even frames could be prevented from referencing odd frames, creating a base 30fps layer and an enhancement 60fps layer.
This is effectively a custom SVC scheme, however it's limited to time (can't change resolution) and the enhancement layer will be significantly smaller than the base layer.

The purpose of these layers is to support degrading the quality of the broadcast.
A subscriber could limit bandwidth usage by choose to only receive the base layer or a subset of the enhancements layers.
During congestion, the base layer can be prioritized while the enhancement layers can be deprioritized/dropped.
However, the cost is a small increase in bitrate (10%) as limiting potential references can only hurt the compression ratio.

When using MoqTransfork, each layer is delivered as a separate Track.
This allows the subscriber to choose which layers to receive and how to prioritize them in SUBSCRIBE.
It also enables layers to be prioritized within the layer application, for example Alice's base layer is more important than Bob's enhancement layer.

The application is responsible for determining the relationship between layers, since they're unrelated tracks as MoqTransport is concerned.
The application could use a catalog to advertise the layers and how to synchronize them, for example based on the Group Sequence.

### Non-Reference Frames
While not explicitly stated, I believe the complexity in MoqTransport stems almost entirely from a single use-case: the ability to drop individual non-reference frames in the middle of a group.

A non-reference frame cannot be referenced by other frames and is effectively a leaf node in the dependency graph.
In theory, non-reference frames can be dropped during congestion without affecting the decodability of other frames.
This idea sounds good on paper but it's a trap.

Good encoders try to avoid non-reference frames because they fundamentally reduce the compression ratio; they are dead-end bits that can't be used for prediction.
You can configure the encoder to produce non-reference frames but the result is a higher bitrate for the ability to drop a small amount during congestion.
But the main problem is the complexity introduced into the transport, as each frame must be transmitted as an individual QUIC stream based on a dependency graph that is not available to relays, and difficult for both broadcasters and viewers to parse.

The ability to drop individual non-reference frames in the middle of a group is an explicit non-goal for MoqTransfork.
An alternative is to put them into a separate layer, such that the tail of the layer could be dropped.


## Audio
Unlike video, audio is simple and yet has perhaps more potential for optimization.

### Frames
Audio samples are very small and for the sake of compression, are grouped into a frame.
This depends on the codec and the sample rate but each frame is typically 10-50ms of audio.

Audio frames are independent and ordered, again making them a good fit for QUIC streams.
Each audio frame can be transmitted as a GROUP with a single FRAME.

### Groups
It may also be desirable to group audio frames into larger units.

For example, if an application wants reliable playback and A/V synchronization, then audio and video could be aligned.
An application could then subscribe to video and audio starting at group X for both tracks, instead of trying to maintain a mapping between the two based on timestamp.
This is quite common in HLS/DASH as there's no reason to subdivide audio segments at frame boundaries.

This can be accomplished with MoqTransfork by using multiple audio FRAMEs within a GROUP.
This limits the ability to drop individual audio frames during congestion (tail drop only) but that's up to the application.

### FEC
Real-time audio applications often use Forward Error Correction (FEC) to conceal packet loss.
Audio frames are a good candidate for FEC given that they are small and independent.

In an ideal world, FEC would be performed by QUIC based on the properties of the hop.
However this is not currently not supported and FEC is left to the application.

In MoqTransfork, each FEC packet is transmitted as a separate GROUP with a single FRAME.
A real-time subscriber issues a `SUBSCRIBE` with an aggressive `Group Expires` value in the milliseconds range.
The publisher will drop any Groups that have not been transmitted or acknowledged within this time frame, potentially causing them to be lost.

The subscriber can choose if it wants to use datagrams or streams for a subscription.
I recommend using streams if the RTT is smaller than `Group Expires`, as it gives retransmissions a chance even when FEC is used.


## Metadata
There's a number of non-media use cases that can be served by MoqTransfork.

### Catalog
Originally part of the transport itself, the catalog is a list of all tracks within a broadcast.
It's since been delegated to the application and is now just another track with a well-known name.

The proposed MoQ catalog format supports live updates.
It does this by encoding a base JSON blob and then applying JSON patches over time.
If the number of deltas becomes too large, the producer can start over with a new base JSON blob.

In MoqTransfork, the base and all deltas are a single GROUP.
The base is the first FRAME and all deltas are subsequent FRAMEs.
The producer can create a new GROUP to start over, repeating the process.

### Timeline
Another track that is commonly pitched is a timeline track.
This records the presentation timestamp of each Group, giving a VOD viewer to seek to a specific time.

The `timeline` track is a single Group containing a Frame for each timestamp.
The live nature of the timeline track is great for DVR applications while being concise enough for VOD.
Timed metadata would use a similar approach or perhaps leverage this track.

### Interaction
Another common use-case is to transmit user interactions, such as controller inputs or chat messages.
It's up to the application to determine the format and encoding of these messages.

Let's take controller input as an example.
The application needs to determine its loss vs latency tolerance, as reordering or dropping inputs will lead to a poor user experience.

- If you don't want loss, then use a single GROUP with a FRAME per input.
- If you don't want latency, then use a GROUP per input with a single FRAME.
- If you want a hybrid, then use form of clustering inputs into GROUPs and FRAMEs based on time.

A publisher could monitor the session RTT or stream acknowledgements to get a sense of the latency and create Groups accordingly.
However, this only applies to the first hop and won't be applicable when relays are involved.

## Latency
One explicit goal of MoqTransfork is to support multiple latency targets.

This is accomplished by using the same Tracks and Group for all viewers, but slightly changing the behavior based on the subscription.
This is driven by the subscriber, allowing them to choose the trade-off between latency and reliability.
This may be done on the fly via SUBSCRIBE_UPDATE, for example if a high-latency viewer wishes to join the stage and suddenly needs real-time latency.

The below examples assume one audio and one video track.
See the next section for more complicated broadcasts.

### Real-Time
Real-time latency is accomplished by prioritizing the most important media during congestion and skipping the rest.

This is slightly different from other media protocols which instead opt to drop packets.
The end result is similar, but prioritization means utilizing all available bandwidth as determined by the congestion controller.
A subscriber or publisher can reset groups to avoid wasting bandwidth on old data.

A real-time viewer could issue:

~~~
SUBSCRIBE track=audio priority=0 order=DESC group_expires=100ms
SUBSCRIBE track=video priority=1 order=DESC group_expires=100ms
~~~

In this example, audio is higher priority than video, and newer groups are higher priority than older groups.
Suppose a viewer fell behind after a burst of congestion and has to decide which groups to deliver next.
This configuration would result in the transmission order:

~~~
GROUP track=audio sequence=102
GROUP track=audio sequence=101
GROUP track=audio sequence=100
GROUP track=video sequence=5
GROUP track=video sequence=4
~~~

The user experience depends on the amount of congestion:

- If there's no congestion, all audio and video is delivered.
- If there's moderate congestion, the tail of the old video group is dropped.
- If there's severe congestion, all video will be late/dropped and some audio groups/frames will be dropped.

The value of `group_expires` is optional.
In this example it means that the publisher automatically resets each group 100ms after they are no longer the latest.
It's recommended to use the maximum jitter buffer size.

### Unreliable Live
Unreliable live is a term I made up.
Basically we want low latency, but we don't need it at all costs and we're willing to skip some video to achieve it.
This is useful for broadcasts where latency is important but so is picture quality.

An unreliable live viewer could issue:

~~~
SUBSCRIBE track=audio priority=0 order=ASC
SUBSCRIBE track=video priority=1 order=DESC group_expires=3s
~~~

This example is different from the real-time one in that audio is fully reliable and delivered in order.
Of course this is optional and up to the application, as it will result in buffering during significant congestion.
If the viewer goes through a tunnel and then comes back online, they won't miss any audio.

A key difference is that our jitter buffer is much larger for video, 3s in this example.
The player will tolerate up to 3s of latency before it starts skipping past video frames.
Note that the `group_expires` value can be increased during buffering by issuing a SUBSCRIBE_UPDATE.

### Reliable Live
Reliable live is another term I made up.
This is when we have a live stream but primarily care about picture quality.
A good example is a sports game where you want to see every frame.

A reliable live viewer could issue:

~~~
SUBSCRIBE track=audio priority=0 order=ASC
SUBSCRIBE track=video priority=0 order=ASC
~~~

This will deliver both audio and video in order, and with the same priority.
The viewer won't miss any content unless the publisher resets a group.
However, this can result in buffering during congestion and provides a similar user experience to HLS/DASH.

### VOD / DVR
Video on Demand (VOD) and Digital Video Recorder (DVR) both involve seeking backwards in a live stream.
MoqTransfork can serve this use-case too, don't worry.

A VOD viewer could issue:

~~~
SUBSCRIBE track=audio priority=0 order=ASC start=345 end=396
SUBSCRIBE track=video priority=0 order=ASC start=123 end=134
~~~

The application is responsible for determining the group sequence numbers based on the desired timestamp.
This could be done via a `timeline` track or out-of-band.

A subscriber will need a specific `end` or else it will download too much data at once, as old media is transmitted at network speed and not encode speed.
It will need to issue an updated SUBSCRIBE to expand the range as playback continues and the buffer depletes.
A subscriber could use SUBSCRIBE_UPDATE, however there are race conditions involved.

A DVR player does the same thing but can automatically support joining the live stream.
It's perfectly valid to specify a `end` in the future and it will behave like reliable live viewer once it reaches the live playhead.

Alternatively, a DVR player could prefetch the live playhead by issuing a parallel SUBSCRIBE at a lower priority.
This would allow playback to immediately continue after clicking the "Go Live" button, canceling or deprioritizing the VOD subscription.

~~~
SUBSCRIBE track=video priority=0 order=ASC start=123 end=134
SUBSCRIBE track=video priority=1 order=DESC
~~~

### Upstream
All of these separate viewers could be watching the same broadcast.
How is a relay supposed to fetch the content from upstream?

MoqTransfork addresses this by providing the publisher's Track Priority and Group Order in the INFO message.
This is the intended behavior for the first hop and dictates which viewers are preferred.

For example, suppose the producer chooses:

~~~
INFO track=audio priority=0 order=DESC
INFO track=video priority=1 order=DESC
~~~

If Alice is watching a VOD and issues:

~~~
SUBSCRIBE track=audio priority=0 order=ASC
SUBSCRIBE track=video priority=0 order=ASC
~~~

If Bob is watching real-time and issues:

~~~
SUBSCRIBE track=audio priority=0 order=DESC
SUBSCRIBE track=video priority=1 order=DESC
~~~

For any congestion on the first mile, then the relay will improve Bob's experience by following the producer's preference.
However any congestion on the last mile will always use the viewer's preference.

A relay should use the publisher's priority/order only when there's a conflict.
If viewers have the same priority/order, then the relay should use the viewer's preference and it can always issue a SUBSCRIBE_UPDATE when this changes.


## Broadcast
A broadcast is a collection of tracks from a single producer.
This usually includes an audio track and/or a video track, but there are reasons to have more than that.

### ABR
Virtually all mass fanout use-cases rely on Adaptive Bitrate (ABR) streaming.
The idea is to encode the same content at multiple bitrates and resolutions, allowing the viewer to choose based on their unique situation.

MoqTransfork unsurprisingly supports this via multiple Tracks, but relies on the application to determine the relationship between them.
This is often done via a `catalog` track that details each track's name, bitrate, resolution, and codec.
This includes how a group in one track corresponds to a group in another track.
A common approach is to use the same Group Sequence number for all tracks, or perhaps utilize a `timeline` track to map between Group Sequences and presentation timestamps.

The viewer may limit the available tracks based on capabilities or preferences.
For example, the device may not support the 4K track since it uses AV1, or the screen size may be too small to justify the bandwidth.
This is easy enough to support; just ignore these tracks in the catalog.

The primary reason to use ABR is to adapt to changing network conditions.
The viewer learns about the estimated bandwidth via the SESSION_UPDATE message, or by measuring network traffic and can then choose the appropriate track based on bitrate.

Transitioning between tracks can be done seamlessly by utilizing prioritization.
For example, suppose a viewer is watching the 360p track and wants to switch to 1080p at group 69.

A real-time or unreliable live viewer could issue:

~~~
SUBSCRIBE_UPDATE track=360p  priority=1 order=DESC end=69
SUBSCRIBE        track=1080p priority=0 order=DESC start=69
~~~

A reliable live or VOD viewer could issue:

~~~
SUBSCRIBE_UPDATE track=360p  priority=0 order=ASC end=69
SUBSCRIBE        track=1080p priority=1 order=ASC start=69
~~~

The difference between them is whether to prioritize the old track or the new track.
In both scenarios, the subscription will seamlessly switch at group 69 even if it's seconds in the future.
The same behavior can be used to switch down.

### SVC
We touched on SVC before, but it's worth mentioning as an alternative to ABR.
I want to see it used more often but I doubt it will be.

Instead of choosing the track based on the bitrate, the viewer subscribes to them all:

~~~
SUBSCRIBE track=360p  priority=0 order=DESC
SUBSCRIBE track=1080p priority=1 order=DESC
SUBSCRIBE track=4k    priority=2 order=DESC
~~~

During congestion, the 4k enhancement layer will be deprioritized followed by the 1080p enhancement layer.
This is a more efficient use of bandwidth than ABR, but it requires more complex encoding.


## Conferences
Some applications involve multiple producers, such as a conference calls or a live events.
Even though these are separate broadcasts from potentially separate origins, MoqTransfork can still serve them over the same session.

### Discovery
The first step to joining a conference is to discover the available broadcasts.

There is currently no discovery mechanism in MoqTransfork.
However, an application can build one on top of a MoqTransfork track (of course!).

For example, suppose we have a conference room called `room.12345`.
An index service could produce a `room.12345` track that lists all broadcasts within the room.
When Alice joins and ANNOUNCES `room.12345.alice`, the index service could update the `room.12345` track to add a new FRAME `+alice`.
The same can be done to remove her when she leaves.

### Participants
Extending the idea that audio is more important than video, we can prioritize tracks regardless of the source.
This works because `SUBSCRIBE priority` is scoped to the session and not the broadcast.

~~~
SUBSCRIBE broadcast=alice track=audio priority=1
SUBSCRIBE broadcast=frank track=audio priority=1
SUBSCRIBE broadcast=alice track=video priority=3
SUBSCRIBE broadcast=frank track=video priority=3
~~~

When Alice starts talking or is focused, we can actually issue a SUBSCRIBE_UPDATE to increase her priority:

~~~
SUBSCRIBE_UPDATE broadcast=alice track=audio priority=0
SUBSCRIBE_UPDATE broadcast=frank track=video priority=2
~~~

Note that audio is still more important than video, but Alice is now more important than Frank. (poor Frank)

This concept can further be extended to work with SVC or ABR:

~~~
SUBSCRIBE broadcast=alice track=360p priority=1
SUBSCRIBE broadcast=frank track=360p priority=2
SUBSCRIBE broadcast=alice track=720p priority=3
SUBSCRIBE broadcast=frank track=720p priority=4
~~~


# Security Considerations
TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
