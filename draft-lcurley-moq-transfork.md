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
#  catalog: I-D.wilaw-moq-catalogformat

--- abstract

TODO Abstract


--- middle

# Fork

This draft is based on moq-transport-03 [moqt].
The concepts, motivations, and terminology are very similar on purpose.
When in doubt, refer to the original draft.

I absolutely believe in the motivation and potential of Media over QUIC.
The layering is phenomenal and addresses many of the problems with current live media protocols.
I fully support the goals of the working group and the IETF process.

However, there are some flaws with MoqTransport that I'd like to address.
It's been years and we're still unable to align on the most critical property of the transport... how to utilize QUIC.
The draft supports multiple different approaches, but it does so by leaving important properties dynamic or undefined.
In our RUSH to standardize a protocol, the QUICR solutions have led to WARP in ideals.

This fork is meant to be constructive; an alternative vision.
However, we've been arguing about some of these issues for years now and I don't expect that will change any time soon.
I'd like to try leading by example, demonstrating that it's possible to simplify the protocol and still support a documented set of use-cases.

Here's a high level overview of the differences between MoqTransport and MoqTransfork:

## Object Model
The object model is an abstraction exposed to the application, outlining how the transport is used.

My biggest complaint with MoqTransport is that the properties of the object model change dynamically based on the publisher's "delivery preference".
This creates a lot of complexity and edge cases, as the meaning of a track/group/object will subtly change at runtime.

The MoqTransfork object model is static and maps directly to QUIC.
MoqTransfork uses a QUIC stream or datagram per Group, depending on the subscription.
This simplification is still able to support all of the use-cases; see the Appendix.

## Subscriber Priority
As the original proponent of producer choosen priorities (send order), I'm ashamed to admit that I was wrong.
Reality is more naunced; both the subscriber and publisher need to work together.

MoqTransfork delegates the priority decision for the last mile to the subscriber.
This is done via a `Track Priority` and `Group Order` field within `SUBSCRIBE`.

If there's a single viewer, then this priority can be used all the way to origin.
However when there are multiple conflicting viewers, then a relay should use the producer's preference during conflicts as advertised in `INFO`.

## Control Streams
At the core of a transport are control messages.

MoqTransport control messages are fine, but have the potential for head-of-line blocking as they share a single stream.
There's also just a lot of messages for state transitions such as `UNSUBSCRIBE`, `SUBSCRIBE_ERROR`, `SUBSCRIBE_DONE`, etc.

MoqTransfork continues the trend of leveraging QUIC with a separate control stream per transaction.
The stream is instead closed or reset with an error code.
This removes the need for any messages that start with `UN` or end with `_DONE`, `_ERROR`, `_RESET`, etc.

## Byte Offsets
When a connection or subscription is severed, it's often desirable to resume where it left off.

MoqTransport implements this via subscriptions that can start/end at an object ID within a group.
This is an okay approach, however it results in redownloading partial objects and a fragmented cache at parsed boundaries.

MoqTransfork instead utilizes FETCH to serve incomplete Groups starting at a byte offset.
QUIC streams are tail dropped when reset so there's no need for a more complex mechanism.
It also allows an advanced relay to forward STREAM frames directly.

## Datagram Support
Datagrams are useful when payloads are small, overhead is important, and the desired latency is below the RTT.

MoqTransport supports sending objects as QUIC datagrams via a publisher track preference.
This works for real-time viewers but results in a poor experience for higher latency viewers.
Datagrams also disable drop notifications which are required for reliable playback.

MoqTransfork instead lets the subscriber indicate if it wants to receive datagrams for a subscription.
If so, Groups may be silently dropped for whatever reason, but it saves a few bytes on the wire (~10 per Group) which is desirable for audio use-cases.

## Use-Cases
The appendix contains a number of media use-cases and recommended approaches.
These are by no means required or comprehensive, but are meant to help illustrate the careful layering of Media over QUIC.


# Conventions and Definitions
{::boilerplate bcp14-tagged}


# Concepts
Many of the concepts are borrowed from MoqTransport so I'm not going to fully rehash them here.
A future draft will.

## Object Model
The MoqTransfork object model consists of:

- **Broadcast**: A collection of Tracks from a single producer.
- **Track**: A series of Groups within a Broadcast.
- **Group**: A series of Frames within a Track, served in order.
- **Frame**: A sized payload of bytes within a Group.

### Broadcast
Each broadcast is identified by a unique name within the session.
A publisher may optionally ANNOUNCE a broadcast or the subscriber can determine the name out-of-band.

The MoqTransport draft refers to this as "track namespace".
I couldn't help but bikeshed.

### Track
Each subscription is scoped to a single track and starts/ends at a Group boundary.
The subscriber chooses the priority/order of each track, dictating which track arrives first during congestion.

Each track has a unique name within the broadcast.

There is currently no way to discover tracks within a broadcast; it must be negotiated out-of-band.
For example, a `catalog` track that lists all other tracks and their association.


### Group
A Group is an ordered stream of bytes within a Track.
It has a small header containing a sequence number within the track and an expiration time.

A Group may be served via a QUIC stream or datagram, depending on the subscription.
If the Group is dropped for any reason, a subscriber may issue a FETCH to resume at a given byte offset.

### Frame
Frames currently only provides framing, hence the name.

Framing is useful in some applications but can also be redundant or increase memory usage, as the size must be known upfront.
This will be removed in a future version of the draft, delegated to a higher layer.


# Flow
This section outlines the flow of messages within a MoqTransfork session.
See the section for Messages section for the specific encoding.

## Establishment
MoqTransfork runs on top of either WebTransport or QUIC.

After a connection is established, the endpoints perform a MoqTransfork handshake to negotiate the version and other parameters.
The client opens the Session Stream and sends a SESSION_CLIENT message and the server replies with a SESSION_SERVER message.

TODO copy this from MoqTransport.

## Bidirectional Streams
Bidirectional streams are primarily used for control streams.

Note that QUIC bidirectional streams have both a send and recv direction can be closed or reset (with an error code) independently.
This is used to indicate completion or errors respectively.

The first varint of each stream indicates the Stream Type.
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
| 0x3  | Datagram  | Subscriber |
|------|-----------|------------|
| 0x4  | Fetch     | Subscriber |
|------|-----------|------------|
| 0x5  | Info      | Subscriber |
|------|-----------|------------|


### Session
The Session stream contains all messages that are session level.

The client MUST open the one Session Stream immediately after establishing the QUIC/WebTransport session.
The client sneds a SESSION_CLIENT message and the server replies with a SESSION_SERVER message.
Afterwards, both endpoints MAY send SESSION_INFO messages containing updated information about the session.

The session remains active until the Session Stream is closed or reset by either endpoint.


### Announce
A publisher MAY open an Announce Stream to advertise a broadcast.
This is optional and the subscriber MAY determine the broadcast name out-of-band.

The publisher MUST start the stream with an ANNOUNCE message.
The subscriber MUST reply with an ANNOUNCE_OK message or reset the stream.
The announcement is active until the stream is closed or reset by either endpoint.

There is currently no expectation that a relay will forward an ANNOUNCE message downstream.
A future draft may introduce a mechanism to discover broadcasts matching a prefix.

### Subscribe
A subscriber MAY open a Subscribe Stream to request a named track within a broadcast.
The publisher will start transmitting Group matching the criteria via QUIC streams.

The subscriber MUST start the stream with a SUBSCRIBE message followed by any number SUBSCRIBE_UPDATE messages.
The publisher MUST reply with an INFO message followed by any number of GROUP_DROPPED messages.

The SUBSCRIBE message contains information about the track and an indicated range.
The publisher MUST transmit a GROUP or GROUP_DROP message for each Group within the range.

The subscription is active until the either endpoint closes or resets the stream.
The subscriber MUST close the subscription when all GROUP and GROUP_DROP messages have been received.
The publisher MAY close the subscription after all groups have been acknowledged.

### Datagram
A subcriber MAY open a Datagram Stream (TODO name) to request a named track within a broadcast.
The publisher will start transmitting any Groups matching the criteria via QUIC datagrams.

The subscriber MUST start the stream with a SUBSCRIBE message and MAY follow with subsequent SUBSCRIBE_UPDATE messages.
The publisher MUST reply with an INFO message and nothing else.

The publisher MAY transmit each Group as a Group Datagram or Frame Datagram.
The publisher MAY drop a Group for any reason, including congestion and MTU constraints.

The subscription is active until the either endpoint closes or resets the stream.
The publisher MUST close the subscription after transmitting all Group/Frame Datagrams.
The subscriber MAY close the subscription when all Group/Frame Datagrams have been received.

## Info
The subscriber MAY open an Info Stream to request information about a track.
This is not necessary for a subscription, as SUBSCRIBE will trigger an INFO reply.

The subscriber MUST start the stream with a INFO_REQUEST message.
The publisher MUST reply with an INFO message or reset the stream.
Both endpoints MUST close the stream afterwards.

## Fetch
A subscriber MAY open a Fetch Stream to receive a single Group at a specified offset.
This is primarily used to recover from an abrupt stream termination, causing the truncation of a Group.

The subscriber MUST start the stream with a FETCH message.
The publisher MUST reply with a GROUP message followed by the Group payload starting at the specified offset.
The fetch is active until both endpoints close the stream, or either endpoint resets the stream.


## Unidirectional
Unidirectional streams are used for subscription data.

|------|--------|-----------|
| ID   | Stream | Role      |
|-----:|:-------|-----------|
| 0x0  | Group  | Publisher |
|------|--------|-----------|

### Group
The publisher creates Group Streams in response to a Subscribe Stream.

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

### Frame Datagram
A Frame Datagram consists of a GROUP message followed by the Frame Payload.
This saves 1-2 bytes by skipping the FRAME header when there's a single Frame.


# Messages
This section covers the encoding of each message.

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

## SESSION_INFO

~~~
SESSION_INFO Message {
  Session Bitrate (i)
}
~~~

**Session Bitrate**: The estimated bitrate of the session in bits per second. This SHOULD be sourced directly from the QUIC congestion controller. A value of 0 indicates that this information is not available.

Sender-side bitrate estimates are important when application-limited, which is common for live media.
A publisher SHOULD transmit a new SESSION_INFO message when the bitrate changes significantly.


## ANNOUNCE
The publisher sends an ANNOUNCE message to advertise a broadcast.

~~~
ANNOUNCE Message {
  Broadcast Name (b),
}
~~~

## ANNOUNCE_OK
The subscriber replies to an ANNOUNCE with an ANNOUNCE_OK message.

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
  Datagrams Enabled (i)
  Group Order (i)
  Group Expires (i)
  Group Min (i)
  Group Max (i)
}
~~~

**Track Priority**: The transmission priority of the subscription relative to all other active subscriptions within the session. The publisher SHOULD transmit *lower* values first during congestion.

**Datagram Enabled**: A boolean (0 or 1) indicating whether the subscriber supports datagrams. If true, the publisher MUST NOT transmit GROUP_DROP messages and MAY transmit any GROUP via a datagram instead of a stream.

**Group Order**: The transmission order of the Groups within the subscription. The publisher SHOULD transmit groups based on their sequence number in default (0), ascending (1), or descending (2) order.

**Group Expires**: The duration in milliseconds *after completion* that a group is considered fresh. The publisher SHOULD reset a GROUP stream after this time has elapsed, including any time spent cached. A value of 0 means there is no explicit expiration.

**Group Min**: The minimum group sequence number plus 1. A value of 0 indicates the latest group as determined by the publisher.

**Group Max**: The maximum group sequence number plus 1. A value of 0 indicates there is no maximum.


## SUBSCRIBE_UPDATE
The subscriber can modify a subscription with a SUBSCRIBE_UPDATE message.

~~~
SUBSCRIBE_UPDATE Message {
  Track Priority (i)
  Group Order (i)
  Group Min (i)
  Group Max (i)
}
~~~

**Group Min**: The new minimum group sequence, or 0 if there is no update. This value MUST NOT be smaller than prior SUBSCRIBE and SUBSCRIBE_UPDATE messages.

**Group Max**: The new maximum group sequence, or 0 if there is no update. This value MUST NOT be larger than prior SUBSCRIBE or SUBSCRIBE_UPDATE messages.


## INFO
The INFO message contains the current information about a track.

~~~
INFO Message {
  Latest Group (i)
  Default Track Priority (i)
  Default Group Order (i)
}
~~~

**Latest Group**: The latest group as currently known by the publisher. A relay without an active subscription SHOULD forward this request upstream

**Default Track Priority**: The default priority of this track within the broadcast. Note that this is different than SUBSCRIBE, which is scoped to a session instead. The publisher SHOULD transmit subscriptions with *lower* values first during congestion.

**Default Group Order**: The default order of the groups within the subscription: none (0), ascending (1), or descending (2).


## INFO_REQUEST
The INFO_REQUEST message is used to request an INFO response.

~~~
INFO_REQUEST Message {
  Broadcast Name (b)
  Track Name (b)
}
~~~


## FETCH
The subscriber can request a byte offset within a Group with a FETCH message.

~~~
FETCH Message {
  Broadcast Name (b)
  Track Name (b)
  Track Priority (i)
  Group Sequence (i)
  Group Offset (i)
}
~~~

**Track Priority**: The priority of the group relative to all other FETCH and SUBSCRIBE requests within the session. The publisher should transmit *lower* values first during congestion.

**Group Offset**: The requested offset in bytes *after* the GROUP message.


## GROUP Message
The GROUP message contains information about a Group, as well as a reference to the subscription being served.

~~~
GROUP Message {
  Subscribe ID (i)
  Group Sequence (i)
  Group Expires (i)
}
~~~

**Group Expires**: A duration in milliseconds. The group has expired if this duration has elapsed after group has finished, including any time spent cached. The publisher SHOULD NOT transmit expired groups and SHOULD reset any expired Group Streams.


## GROUP_DROP
The publisher transmits a GROUP_DROP message when it is unable to serve a group for a SUBSCRIBE.

~~~
GROUP_DROP {
  Group Sequence Start (i)
  Group Sequence Count (i)
  Group Error Code (i)
}
~~~


**Group Sequence Start**: The sequence number for the first group within the dropped range.

**Group Sequence Count**: The number of additional groups after the first. This value is 0 when only one group is dropped.

**Error Code**: An error code indicated by the application.


## FRAME Message
The FRAME message consists of a length followed by that many bytes.

~~~
FRAME Message {
  Payload (b)
}
~~~

**Payload**: An application specific payload.
A MoqTransfork library MUST NOT inspect or modify the contents unless otherwise negotiated.


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
We can ignore the complexity of P/B frames and focus on I-Frames.

Each I-Frame begins a Group of Pictures (GoP).
A GoP is a set of frames that MAY depend on each other and MUST NOT depend on other GoPs.
Additionally, each GoP has a decode order (DTS) and a frame MUST NOT depend on frames with a higher DTS.

This perfectly maps to a QUIC stream, as they too are independent and ordered.
The easiest way to use MoqTransfork is to send each GoP as a GROUP with each frame as a FRAME, hence the names.

Given the nature of QUIC streams and GoPs, all actions are done at Group boundaries.
A subscription starts at a Group ensuring that it starts with an I-Frame.
Each Group is delivered in decode order ensuring that all frames are decodable (no artifacts).

A subscriber can choose the Group Order based on the desired user experience:

- `SUBSCRIBE group_order=ASC` delivers old Groups to prevents skipping. intended for VOD and reliable live streams.
- `SUBSCRIBE group_order=DESC` delivers new Groups to allow skipping, intended for low-latency live streams.

A publisher or subscriber can skip the remainder of a Group by resetting the GROUP stream.
This truncates the Group but all data received up until that point is decodable.
The subscriber can issue a FETCH if it wants to resume at a byte offset.

### Layers
An advanced application can subdivide a GoP into layers.

The most comprehesive way to do this is with Scalable Video Coding (SVC).
There is a base layer and one or more enhancement layers that depend on lower layers.
For example, a 4K stream could be broken into 4K, 1080p, and 360p (base) layers.
However, SVC has limited support and is complex to encode.

A better supported approach is to use temporal scalability via something like B-pyramids.
The frames within a GoP are sub-divided based on their dependencies, intentionally creating a hierarchy.
For example, even frames could be prevented from referencing odd frames, creating a base 30fps layer and an enhancement 60fps layer.
This is effectively a custom SVC scheme, however it's limited to time (can't change resolution) and the enhancement layer will be significantly smaller than the base layer.

The purpose of these layers is to support degrading the quality of the broadcast.
A subscriber could choose to only receive the base layer, or a subset of the enhancements layers, to limit bandwidth usage.
During congestion the enhancement layers can be deprioritized and dropped, while leaving the base layer intact.
However, the cost is a marginal increase in bitrate (10%) as limiting potential references can only hurt the compression ratio.

With MoqTransfork, each layer is a separate track.
This allows the subscriber to choose which layers to receive and how to prioritize them.
It also enables layers to be prioritized within the layer application, for example Alice's base layer is more important than Bob's enhancement layer.

The application is responsible for determining the relationship between layers, since they're unrelated tracks as MoqTransport is concerned.
The application could use a catalog to advertise the layers and how to synchronize them, for example based on the Group Sequence.

### Non-Reference Frames
While not explicitly stated, I believe the complexity in MoqTransport stems almost entirely from a single use-case: the ability to drop individual non-reference frames in the middle of a group.

A non-reference frame is a leaf node in the dependency graph, as it's not used as a reference by any other frame.
In theory, non-reference frames can be dropped during congestion without affecting the decodability of other frames.
This idea sounds good on paper but it's a trap.

Good encoders try to avoid non-reference frames because they fundamentally reduce the compression ratio.
They are dead-end bits that can't be used for prediction.
You can configure the encoder to produce non-reference frames but the net result is a ~10% increase in bitrate (or cooresponding drop in quality).
This is not worth the ability to halve the frame rate during congestion and drop the bitrate by only ~20%.

This is an explicit non-goal for MoqTransfork.
The benefits are marginal to non-existant and the complexity introduced is high.


## Audio
Unlike video, audio is simple and yet has perhaps more potential for optimization.

### Frames
Audio samples are very small and for the sake of compression, are grouped into a frame.
This depends on the codec and the sample rate but each frame is typically 10-50ms of audio.

Audio frames are independent and ordered, again making them a good fit for QUIC streams.
Each audio frame can be transmitted as a single FRAME within a GROUP.

### Groups
It may also be desirable to group audio frames into larger units.

For example, if an application wants reliable playback and A/V synchronization, then audio and video could be aligned.
An application could then subscribe to video and audio starting at group X for both tracks, instead of trying to maintain a mapping between the two based on timestamp.
This is quite common in HLS/DASH as there's no reason to subdivide audio segments at frame boundaries.

This can be accomplished with MoqTransfork by using multiple audio FRAMEs within a GROUP.
This limits the ability to drop audio during congestion (tail-drop only), but that's up to the application.

### FEC
Real-time audio applications often use Forward Error Correction (FEC) to conceal packet loss.
Audio frames are a good candidate for FEC given that they are small and independent.

In an ideal world, FEC would be performed by QUIC based on the properties of the hop.
However this is not currently not supported and FEC is left to the application.

In MoqTransfork, each FEC frame is transmitted as a separate GROUP with a single FRAME.
A real-time subscriber issues a `SUBSCRIBE` with an aggressive `Group Expires` value in the milliseconds range.
The publisher will reset any Groups that have not been acknowledged within this time frame, potentially causing them to be lost.
This does incur additional overhead in the form of RESET_STREAM and the corresponding GROUP_DROP message.

However, the purpose of this design is to also support higher latency subscribers.
A VOD or higher latency subscriber can issue a `SUBSCRIBE` with a conservative or no `Group Expires` values.
The downside is that the audio track will be higher bitrate due to the unnecessary FEC frames.

A future version of the draft may enable serving a track via datagrams.
An advanced application could potentially have two audio tracks: one with FEC datagrams for real-time and another with reliable streams for higher latencies.
However I think experimentation is needed.

## Metadata
There's a number of non-media use cases that can be served by MoqTransfork.

### Catalog
Originally part of the transport itself, the catalog is a list of all tracks within a broadcast.
It's since been delegated to the application and is now just another track with a well-known name.

The proposed MoQ catalog format [catalog] supports live updates.
It does this by encoding a base JSON blob and then applying JSON patches over time.
If the number of deltas becomes too large, the producer can start over with a new base JSON blob.

In MoqTransfork, the base and all deltas are a single GROUP.
The base is the first FRAME and all deltas are subsequent FRAMEs.
The producer can create a new GROUP to start over, repeating the process.

### Interaction
Another common use-case is to transmit user interactions, such as controller inputs or chat messages.
It's up to the application to determine the format and encoding of these messages.

Let's take controller input as an example.
The application needs to determine its loss vs latency tolerance, as reoredring or dropping inputs will lead to a poor user experience.

- If you don't want loss, then use a single GROUP with a FRAME per input.
- If you don't want latency, then use a GROUP per input with a single FRAME.
- If you want a hybrid, then use form of clustering inputs into GROUPs based on time.

The publisher MAY monitor the session RTT or stream acknowledgements to get a sense of the latency.
It could then use this information to create new GROUPs or reset existing ones on demand.
However this only applies to the first hop and won't be applicable when relays are involved.

## Latency
One explicit goal of MoqTransfork is to support multiple latency targets.

This is accomplished by using the same Tracks and Group for all viewers, but sliiightly changing the behavior based on the subscription.
This is driven by the subscriber, allowing them to choose the tradeoff between latency and reliability.
This may be done on the fly, for example if a high-latency viewer wishes to join the stage and suddenly needs real-time latency.

The below examples assume one audio and one video track.
See the next section for more complicated broadcasts.

### Real-Time
Real-time latency is accomplished by prioritizing the most important media during congestion and skipping the rest.

This is slightly different from other media protocols which instead opt to drop packets.
The end result is similar, but prioritization will result in utilizing all available bandwidth as determined by the congestion controller.
The subscriber or publisher can reset groups to avoid wasiting bandwidth on old data.

A real-time viewer could issue:
```
SUBSCRIBE track=audio track_priority=0 group_order=DESC group_expires=100ms
SUBSCRIBE track=video track_priority=1 group_order=DESC group_expires=100ms
```

In this example, audio is higher priority than video, and newer groups are higher priority than older groups.
Suppose a viewer fell behind after a burst of congestion and has to decide which groups to deliver next.
This configuration would result in the order:

```
GROUP track=audio sequence=102
GROUP track=audio sequence=101
GROUP track=audio sequence=100
GROUP track=video sequence=5
GROUP track=video sequence=4
```

The result depends on the amount of congestion:

- If there's no congestion, all audio and video is delivered.
- If there's moderate congestion, the tail of the old video group (4) is dropped.
- If there's severe congestion, some audio groups/frames will be dropped (100) and all video will late/dropped.

The `group_expires` is optional, and in example it means that the publisher automatically resets each group 100ms after they are no longer the latest.
It's recommended to use the maximum jitter buffer size.

### Unreliable Live
Unreliable live is a term I made up.
Basically we want low latency, but we don't need it at all costs and we're willing to skip some video to achieve it.
This is useful for broadcasts where latency is important but so is picture quality.

An unreliable live viewer could issue:
```
SUBSCRIBE track=audio track_priority=0 group_order=ASC
SUBSCRIBE track=video track_priority=1 group_order=DESC group_expires=3s
```

This example is different from the real-time one in that audio is fully reliable and delivered in order.
Of course this is optional and up to the application, as it will result in buffering during significant congestion.
If the viewer goes through a tunnel and then comes back online, they won't miss any audio.

But the key difference is that our jitter buffer is much larger for video, 3s in this example.
The player will tolerate up to 3s of latency before it starts skipping past video frames.
Note that the `group_expires` value can be increased (buffering) by issuing a SUBSCRIBE_UPDATE.

### Reliable Live
Reliable live is another term I made up.
This is when we have a live stream but primarily care about picture quality.
A good example is a sports game where you want to see every frame.

A reliable live viewer could issue:
```
SUBSCRIBE track=audio track_priority=0 group_order=ASC
SUBSCRIBE track=video track_priority=0 group_order=ASC
```

This will deliver both audio and video in order, and with the same priority.
The viewer won't miss any content unless the publisher resets a group.
However, this can result in buffering during congestion and provides a similar user experience to HLS/DASH.

### VOD/DVR
Video on Demand (VOD) and Digital Video Recorder (DVR) both involve seeking backwards in a live stream.
MoqTransfork can serve this use-case too, don't worry.

A VOD viewer could issue:
```
SUBSCRIBE track=audio track_priority=0 group_order=ASC group_start=345 group_end=396
SUBSCRIBE track=video track_priority=0 group_order=ASC group_start=123 group_end=134
```

The application is responsible for determining the group sequence numbers based on the desired timestamp.
This could be done via a `timeline` track or out-of-band.

The subscriber will need a specific `group_end` or else it will download too much data at once, as old media is transmitted at network speed and not encode speed.
It will need to issue an updated SUBSCRIBE to expand the range as playback continues and the buffer depletes.
The subscriber could use SUBSCRIBE_UPDATE, however there are race conditions involved.

A DVR player does the same thing but can automatically support joining the live stream.
It's perfectly valid to specify a `group_end` in the future and it will behave like reliable live viewer once it reaches the live playhead.

Alternatively, a DVR player could prefetch the live playhead by issuing a parallel SUBSCRIBE at a lower priority.
This would allow playback to immediately continue after clicking the "Go Live" button, cancelling or deprioritizing the VOD subscription.

```
SUBSCRIBE track=video track_priority=0 group_order=ASC group_start=123 group_end=134
SUBSCRIBE track=video track_priority=1 group_order=DESC
```

### Upstream
All of these separate viewers could be watching the same broadcast.
How is a relay supposed to fetch the content from upstream?

MoqTransfork addresses this by providing a Default Track Priority and Default Group Order in the INFO and SUBSCRIBE_OK message.
This is the intended behavior for the first hop and dictates which viewers are preferred.

For example, suppose the producer chooses:
```
INFO track=audio default_track_priority=0 default_group_order=DESC
INFO track=video default_track_priority=1 default_group_order=DESC
```

If viewer Alice is watching a VOD and issues:
```
SUBSCRIBE track=audio track_priority=0 group_order=ASC
SUBSCRIBE track=video track_priority=0 group_order=ASC
```

If viewer Bob is watching real-time and issues:
```
SUBSCRIBE track=audio track_priority=0 group_order=DESC
SUBSCRIBE track=video track_priority=1 group_order=DESC
```

For any congestion on the first mile, then the relay will improve Bob's experience by following the producer's preference.
Note that any congestion on the last mile will always use the viewer's preference.

A relay should use the default priority/order only when there's a conflict.
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
For example, the device may not support the 4K track since it uses AV1, or the window size may be too small to justify the bandwidth.
This is easy enough to support; just ignore these tracks in the catalog.

The primary reason to use ABR is to adapt to changing network conditions.
The viewer learns about the estimated bandwidth via the SESSION_INFO message, or by measuring network traffic, and can then choose the appropriate track based on bitrate.

Transitioning between tracks can be done seamlessly by utilizing prioritization.
For example, suppose a viewer is watching the 360p track and wants to switch to 1080p at group 69.

A real-time or unreliable live viewer could issue:
```
SUBSCRIBE_UPDATE track=360p  track_priority=1 group_order=DESC group_end=69
SUBSCRIBE        track=1080p track_priority=0 group_order=DESC group_start=69
```

A reliable live or VOD viewer could issue:
```
SUBSCRIBE_UPDATE track=360p  track_priority=0 group_order=ASC group_end=69
SUBSCRIBE        track=1080p track_priority=1 group_order=ASC group_start=69
```

The difference between them is whether to prioritize the old track or the new track.
In both scenarios, the subscription will seamlessly switch at group 69 even if it's seconds in the future.
The same behavior can be used to switch down.

### SVC
We touched on SVC before, but it's worth mentioning as an alternative to ABR.
I want to see it used more often but I doubt it will be.

Instead of choosing the track based on the bitrate, the viewer subscribes to them all:
```
SUBSCRIBE track=360p  track_priority=0 group_order=DESC
SUBSCRIBE track=1080p track_priority=1 group_order=DESC
SUBSCRIBE track=4k    track_priority=2 group_order=DESC
```

During congestion, the 4k enhancement layer will be deprioritized followed by the 1080p ehancement layer.
This is a more efficient use of bandwidth than ABR, but it requires more complex encoding.

### Metadata
Broadcasts also include metadata, such as subtitles, messages, or even advertisements.
These are separate tracks, just like audio and video, and can be selected/prioritized based on the viewer's preference.


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
This works because `SUBSCRIBE track_priority` is scoped to the session and not the broadcast.

```
SUBSCRIBE track=alice.audio track_priority=1
SUBSCRIBE track=frank.audio track_priority=1
SUBSCRIBE track=alice.video track_priority=3
SUBSCRIBE track=frank.video track_priority=3
```

When Alice starts talking or is focused, we can actually issue a SUBSCRIBE_UPDATE to increase her priority:

```
SUBSCRIBE_UPDATE track=alice.audio track_priority=0
SUBSCRIBE_UPDATE track=alice.video track_priority=2
```

Note that audio is still more important than video, but Alice is now more important than Frank. (poor Frank)

This concept can further be extended to work with SVC or ABR:
```
SUBSCRIBE track=alice.360p track_priority=1
SUBSCRIBE track=frank.360p track_priority=2
SUBSCRIBE track=alice.720p track_priority=3
SUBSCRIBE track=frank.720p track_priority=4
```


# Security Considerations
TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
