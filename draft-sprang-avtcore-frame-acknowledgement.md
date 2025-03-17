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
title: "RTCP Feedback Message and Request Mechanism for Frame-level Acknowledgement"
abbrev: "Video Frame Acknowledgement"
category: info

docname: draft-sprang-avtcore-frame-acknowledgement-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: AREA
workgroup: avtcore WG
keyword:
 - RTP
 - RTCP
 - Header Extension
 - Video
 - LTR
venue:
  group: avtcore WG
  type: Working Group
  mail: avt@ietf.org
  arch: https://datatracker.ietf.org/wg/avtcore
  github: sprangerik/frame-acknowledgement
  latest: https://github.com/sprangerik/frame-acknowledgement/blob/main/draft-ietf-avtcore-frame-acknowledgement.md

author:
 -
    fullname: Erik Språng
    organization: Google
    email: sprang@google.com

normative:
  DD:
    target: https://aomediacodec.github.io/av1-rtp-spec/#dependency-descriptor-rtp-header-extension
    title: Dependency Descriptor RTP Header Extension
    author:
      org: AOM
  IANARTCP:
    target: https://www.iana.org/assignments/rtp-parameters/rtp-parameters.xhtml#rtp-parameters-4
    title: FMT Values for RTPFB Payload Types
    author:
      org: IANA
  IANAEXT:
    target: https://www.iana.org/assignments/rtp-parameters/rtp-parameters.xhtml#rtp-parameters-10
    title: RTP Compact Header Extensions
    author:
      org: IANA

informative:
  SVC:
    target: https://www.w3.org/TR/webrtc-svc
    title: Scalable Video Coding (SVC) Extension for WebRTC
    author:
      org: W3C
  TWCC:
    target: https://datatracker.ietf.org/doc/html/draft-holmer-rmcat-transport-wide-cc-extensions-01
    title: RTP Extensions for Transport-wide Congestion Control
  VFTI:
    target: http://www.webrtc.org/experiments/rtp-hdrext/video-frame-tracking-id
    title: Video Frame Tracking Id
  LNTF:
    target: https://www.ietf.org/archive/id/draft-majali-avtcore-lntf-feedback-message-00.html
    title: RTCP feedback Message for Loss Notification
    author: Shridhar Majali


--- abstract

This document describes a mechanism for signaling which video frames have been received and decoded by a remote peer. It comprises an RTCP feedback message and an RTP header extension used to request said feedback.

One of the main use cases for this data is to implement various forms of Long Term Reference (LTR) reference structures.

--- middle

# Introduction

The most common way for realtime video to be transmitted is to encode a pretty much fixed scalability structure, such as those in the W3C {{SVC}} Scalability mode list.

In such a scenario, the video encoder produces frames "blindly" without real knowledge of what state the remote receiver is in. Using recovery mechanisms such as retransmission, forward error correction and fast-forwarding past skippable frames the receiver is assumed to be able to decode the video. In some cases those methods may not be enough, requiring keyframe requests to be sent as a last resort.

On the other hand, if the encoder is able to reason about which frames have been received and decoded it can be more proactive. One way is to store frames that are known to be received so that they can be later used as guaranteed good references in the case of e.g. large loss events, avoiding the need for potentially large retransmissions etc. Collectively this is often referred to as "Long Term Reference" structures or LTR for short, although the exact structure may vary.

In order to achieve this the sender must be able to reason about the state of the receiver, necessitating the need for feedback signals. In this document a new RTCP message called "Frame Acknowledgement" is introduced as a codec agnostic feedback message for this purpose. Further, an RTP header extension is introduced that allows the sender to actively request feedback on decoding of the associated frame. This allows the sender to both request quick feedback on frames that are important for latency, and enables resilience against loss of feedback packets.

Note that it is allowed to report a frame as decoded even if the decode process is not complete - as long as the receiver guarantees that it will attempt to decode the frame. The rationale for this is that we want to reduce the feedback delay as much as possible. Should the decoding of a frame that has been acknowledged fail, then the receiver MUST request a keyframe to recover, even if the failed decoding belongs to a droppable layer.

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Applicability

Frame Acknowledgement can be used for video streams in most topologies. It is also designed to be codec agnostic.

In terms of {{?RFC7667}}, Point-to-Point is the most straightforward target as it is easiest to reason about a single receiver. A Media Translator or other systems that include a decoder are similarly easy - from the perspective of the sender the middle box is the receiver.

If a Transport Translator is used for Point-to-Multi-Point, then the middlebox must make sure to make valid translations - e.g. by acknowledging a frame only if all recipients of that stream have acknowledged said frame.

# Existing Feedback Formats

This section provides an overview, for informational purposes, of some existing feedback formats that could be seen as alternatives.

NACK, defined in {{?RFC4585}}, provides only requests for packets the receiver is interested in having retransmitted. Absence of feedback is a poor signal for acknowledgement, especially since said feedback can be lost.

{{?RFC8888}} and {{TWCC}} provide per-packet acknowledgement and so are more useful. A mapping from packet(s) to frame needs to happen but that is not a big problem. However, even if a frame is confirmed to be received there is no guarantee that it gets decoded.

Reference Picture Selection Indication (RPSI) is another existing message, but it puts the logic of requesting a particular reference frame in the receiver - significantly complicating the system especially in Point-to-Multi-Point systems. It is further codec specific, and several modern codecs lack a specification - including AV1 and H.266. 

Loss Notification {{LNTF}} was a proposed RTCP message intended to solve most of these problems, but it lacks resilience against loss of feedback and also cannot handle out-of-order acknowledgements. The latter makes for instance single-SSRC simulcast structures (e.g. SxTx modes in {{SVC}}) impossible.

# Requirements

The messages in this proposal are intended to fulfill the following requirements:

1. Codec agnostic
  The protocol should be general enough to work across all current and future codecs.
2. Payload Invariant
  The protocol should not depend on data within the encoded bitstream payload. That includes codec specific frame identifiers, feedback requests and feedback messages.
3. Uses Frame Identifiers
  Explicit marking of frames, rather than using an indirection via packets.
4. Order Invariant
  The format should not make assumptions about the required decode order of frames.
5. Send-side Controlled
  The sender explicitly indicates when and for which frames feedback should be sent.
6. Loss Resilient
  The sender should be able to detect and recover from lost feedback messages.
7. Low Delay
  The latency should be small, with the sender being able to tune delay vs rate tradeoff.
8. Low Overhead
  The network overhead in terms of both packet rate and bitrate should be minimized.

# Frame Acknowledgment

## Frame identifier selection

In order to request and receive information about decoded frames, we must be able to identify them. Rather than adding new metadata for this purpose alone, we do that by picking the first available option from a list of available sources:

1. The frame_number from a Dependency Descriptor header extension {{DD}}

Note: In this draft version, only a single source is allowed. Future versions may add other alternatives. Cases can be made for anything from a new dedicated identification system (similar to {{VFTI}}) to mappings from codec specific payload data.

## Frame Acknowledgment Request

A Frame Acknowledgement Request is an RTP header extension indicating the oldest frame ID the sender is interested in receiving feedback for.
The request MUST be done on the media SSRC of video frames in question.
The request implies a status request for all frames starting at the given frame ID, up to and including the frame contained in the RTP packet the header extension is attached to - even if that frame is not yet complete.
If the extension is attached to a packet not containing a video frame, the feedback should be up to and including the immediately preceding frame ID.

Note that the Frame ID is a 16 bit counter with rollover, so e.g. a request with Frame ID = 65535 attached to a packet containing Frame ID = 1 is a request for the three frames {65535, 0, 1}.

If a new Frame Acknowledgement Request is sent with an incremented Frame ID, all status values prior to that Frame ID are considered as acknowledged and can be culled by the receiver. A sender MUST NOT request prior to either the last acknowledged Frame ID or start of the stream.

### Data layout overview

     0                   1                   2
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |  ID   | len=1 |           Frame ID            |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

#### Frame ID (16 bits)

The earliest Frame ID that feedback is requested for.

## Frame Acknowledgment

### Data layout overview

Short feedback message (L = 0):
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |V=2|P| FMT=12  |   PT = 205    |          length               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                 SSRC of RTCP packet sender                    |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |        Start Frame ID         |L|   length    |  status + pad |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |  ...                                                          |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Long feedback message:
     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |V=2|P| FMT=12  |   PT = 205    |          length               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                 SSRC of RTCP packet sender                    |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |        Start Frame ID         |L|          length             |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                            status + pad . . .                 |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

#### Start Frame ID (16 bits)

The first Frame ID in this feedback.

#### L (1 bit)

If the number frames in the feedback vector is < 128, then L = 0.
Otherwise L = 1.

#### length (7 bits or 15 bits)

An unsigned integer denoting how many consecutive frames this message contains feedback for. The last Frame ID is thus Frame ID + length - 1.

If L = 0, length is 7 bits, otherwise length is 15 bits.

#### status (N bit)

A bit vector of the length specified in the length field above. For each bit position, the Frame ID is incremented by one.

A value of 0 indicates the frame has not been received and decoded.
A value of 1 indicates the frame has been received and decoded.

# Security Considerations

The messages in this proposal may expose a small amount of data, namely the number of frames that have been sent, and potentially in an indirect way which frames the sender sees as important for recovery.

This data should however not pose any significant privacy or security risks.

# IANA Considerations

The RTP header extension needs to have a URI identifier assigned by IANA. See {{IANAEXT}}.

The RTCP message uses PT = 205 (RTPFB, Generic RTP Feedback). As of writing, the next available FMT value is 12. A dedicated ID needs to be assigned by IANA. See {{IANARTCP}}.

--- back

# Acknowledgments
{:numbered="false"}

