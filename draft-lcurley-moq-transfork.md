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


--- abstract

TODO Abstract


--- middle

# Fork
As one of the authors of the MoqTransport ([moqt]) draft, it is counter-intuitive to fork my own draft.
However, I think its the most constructive way forward.
Hear me out.

## Motivation
The Media over QUIC working group contains some amazing ideals and ideas.
There's been a lot of organic growth and we will eventually produce something special.
I fully support the goals of the working group and the IETF process.

However, we're in the precarious position of developing a protocol via committee for a technology still in its infacy.
There's virtually no implementations in production and yet we're trying to standardize something.
Most of our discussions are theoretical and it's become very difficult (personally) to make progress.

In our RUSH to standardize a protocol, the QUICR solutions often lead to WARP in ideals.

## Critique
A quick critique of the MoqTransport draft.

### Object Model
The MoqTransport draft introduces the concept of Objects, Groups, and Tracks.
While these are generic terms and there's no explicit media mapping on purpose, the implicit intent is that:

- **Object**: Audio/Video Frame
- **Group**: Group of Pictures (video only)
- **Track**: Audio/Video Track

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

## Differences
This is a summary of the high-level differences between the MoqTransport draft ([moqt]) and this draft.

## Object Model
The MoqTF object model inherits the QUIC object model, rather than fighting against it.

The same terms are used for consistency: Object, Group, Track.
The one addition is Broadcast, which was implied in the MoqTransport draft as "namespace".

- **Broadcast**: A collection of tracks, identified by a unique name.
- **Track**: A sequential collection of groups, identified by a unique name.
- **Group**: A sequential collection of objects, served via a QUIC stream.
- **Object**: A sized payload of bytes.

These are not intended to map directly to media concepts.
An Object might consist of multiple frames (ex. moof) or a parital frame (ex. slice).
A Group might consist of non-dependent (ex. audio, multiple GoPs) or a layer within a GoP (ex. SVC).
See the Appendix for an exhaustive list of examples.

## Properties
The object model leverages QUIC to provide concrete properties.

### Broadcast
- **Addressable** via a unique string.
- **Terminal** via STREAM_FIN on an announce stream.
- **Cancelable** via RESET_STREAM on an announce stream.

### Track
- **Addressable** via a unique string within the broadcast.
- **Terminal** via STREAM_FIN on a subscribe stream.
- **Cancelable** via RESET_STREAM on a subscribe stream.
- **Prioritized** via subscribe parameter.
- **Seekable** via subscribe parameters.

### Group
- **Addressable** sequence number within a track.
- **Unordered** within a track.
- **Terminal** with a clean close.
- **Tail Dropable** with an error code.
- **Prioritized** based on the subscription.
- **Flow Controlled** according to QUIC.
- **Expires** after an optional duration.

### Object
- **Addressable** via byte offsets within a group.
- **Sized** with a length prefix.
- **Payload** of opaque bytes.
- **Ordered** within a group.
- **Reliable** via QUIC retransmits.


Some notible properties that were removed:
- **Gaps**: There can be no gaps between groups or objects. Applications can instead signal gaps within the payload (ex. timestamp).
- **Object IDs**: Objects are now addressed by byte offsets within a group. This makes relays more efficient as they no longer need to parse/buffer the stream after the header. It also allows fetching partial objects, like the remaining half of an I-frame.
- **Independent Objects**: Groups are delivered via an ordered/reliable QUIC stream, so objects within a group inherit that behavior. If objects should be delivered/prioritized independently, then they need to be in separate groups (ex. audio) or tracks (ex. SVC). Use the object model based on the properties you want, rather than dynamically change the properties of the object model.


{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
