---
title: "RTCP Feedback Message and Request Mechanism for Frame-level Acknowledgement"
abbrev: "Video Frame Acknowledgement"
category: info

docname: draft-sprang-avtcore-frame-acknowledgement-00
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
  latest: https://github.com/sprangerik/frame-acknowledgement/blob/main/draft-sprang-avtcore-frame-acknowledgement.md

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

# Frame Acknowledgment Extension

The Frame Acknowledgement extension is an RTP header extension used both to identify frames and request feedback about the remote state.
It SHOULD appear on the last packet of a video frame, and MUST NOT appear more than once on a single frame.

## Frame Identifier

In order to request and receive information about decoded frames, we must be able to identify them. The frame acknowledgement header extension may contain a Frame ID field for this purpose. The Frame ID is an 8-bit unsigned integer field, that wraps around to 0 on overflow.

## Frame Acknowledgment Request

In order to get feedback on the state of the remote decoder, the sender actively requests such feedback using the same frame acknowledgement header extension that is also used for frame identification.
The feedback request comprises a start Frame ID and length field. Specifying the range explicitly has several adavantages, including enabling reliable delivery of the feedback since the sender can effectively make retransmision requests of the feedback.

If a new Frame Acknowledgement Request is sent with an incremented Feedback Start, all status values prior to that Frame ID are considered as acknowledged and can be culled by the receiver. A sender MUST NOT request feedback prior to either the last acknowledged Frame ID or start of the stream.

## Frame Acknowledgment Feedback

The actual feedback response is sent as an RTCP message. It contains a sequence of status symbols corresponding to the frame status for the given frame ID vector of a request.

### Data layout overview

In this layout, the first line assumes a One-Byte RTP header extension. The length N will be between 2 and 4, depending on the header values.
The message can trivially be sent using Two-Byte RTP header extensions as well.

     0
     0 1 2 3 4 5 6 7
    +-+-+-+-+-+-+-+-+
    |  ID   | len=N |
    +-+-+-+-+-+-+-+-+
    |FFR| Reserved  |  : Header (REQUIRED)
    +-+-+-+-+-+-+-+-+
    |   Frame ID    |
    +-+-+-+-+-+-+-+-+
    | Feedb. Start  |
    +-+-+-+-+-+-+-+-+
    | Feedb. Length |
    +-+-+-+-+-+-+-+-+

#### FFR (2 bits)

Indicates the contents of this header, with the following possible values:

  00: Frame ID only
      The header byte is followed by a one byte Frame ID field (N = 2). No feedack is requested.
  01: Frame ID + implicit feedback request
      The header byte is followed by a one byte Frame ID field (N = 2). Feedback is requested for the same Frame ID.
  10: Feedback request only
      The header byte is followed by two bytes (N = 3): a one byte Feedback Start and a one byte Feedback Length field. Feedback is requested for frame IDs spanning (Feedback Start) to (Feedback Start + Feedback Length), exclusive. 
  11: Frame ID + independent feedback request
      The header byte is followed by three bytes: a one byte Frame ID, a one byte Feedback Start and a one byte Feedback Length field. This implies both a frame marking and an independent feedback request (the requested span does not need to include the marked frame).

The remaining 6 bits of the header byte are reserved and should be set to 0.

### Frame ID (8 bits)

Present only if FFR is one of 00, 01 or 11.

An unsinged integer that uniquely idefntifies a frame, and MUST be incremented by one for each new frame (in sending order) that needs to be identified.

### Feedback Start (8 bits)

An unsigned integer that corresponds to the first Frame ID the sender is requesting feedback for.

### Feedback Length (8 bits)

An unsigned integer that indicates the number of frames the sender is requesting feedback for.

Note that since the Frame ID wraps around, a request with Freedback Start = 254 and Feedback Length = 3 indicates the sender is requesting feedback for frames with Frame ID 254, 255 and 0.

If a sender is not interested in feedback for frames prior to and including a given Frame ID, it can request feedback with that Frame ID set as Feedback Start, and Feedback Length set to zero.

## Frame Acknowledgment Feedback

The Frame Acknowledgement Feedback message is an RTCP message containing a vector of status symbols, corresponding to the state for the frames requested in Frame Acknowledgement Extension.

### Data layout overview

     0                   1                   2                   3
     0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |V=2|P| FMT=12  |   PT = 205    |          length               |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                     SSRC of packet sender                     |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |                      SSRC of media source                     |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    |Start Frame ID |    Length     |        status + pad ...       |
    +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

#### Start Frame ID (8 bits)

The first Frame ID in this feedback.

#### Length (8 bits)

An unsigned integer denoting how many consecutive frames this message contains feedback for. The last Frame ID is thus Frame ID + Length - 1.

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

