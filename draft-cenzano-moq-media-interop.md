---
title: "MoQ Media Interop"
abbrev: "moq-mi"
category: info

docname: draft-cenzano-moq-media-interop-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
 - Media over QUIC
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "afrind/draft-cenzano-media-interop"
  latest: "https://afrind.github.io/draft-cenzano-media-interop/draft-cenzano-moq-media-interop.html"

author:
 -
    fullname: Jordi Cenzano-Ferret
    organization: Meta
    email: jcenzano@meta.com

 -
    fullname: Alan Frindell
    organization: Meta
    email: afrind@meta.com

normative:
  MOQT: I-D.ietf-moq-transport

  ISO14496:
    target: https://www.iso.org/standard/74429.html
    title: Carriage of network abstraction layer (NAL) unit structured video in the ISO base media file format
    author:
      org: ISO
    date: 2019

informative:
  LOC: I-D.mzanaty-moq-loc

--- abstract

This protocol can be used to send and receive video and audio over Media over
QUIC Transport [MOQT].

--- middle

# Introduction

This protocol specifies a simple mechanism for sending media (video and audio)
over MOQT for both live-streaming and VC use cases.  The protocol is flexible in
order to support this range of use cases.

Media parameters can be updated in the middle of a the track (ex: frame rate,
resolution, codec, etc)

The protocol defines a low overhead packaging format optimized for WebCodecs
called WCP that is extensible to other formats such as FMP4.  This is not LoC
[LOC], but will eventually be merged with that specification.

# Protocol Operation

## Track Names

The publisher selects a namespace of their choosing, and sends an ANNOUNCE
message for this namespace.  Since MoQT tracks are immutable, each new broadcast
MUST have a unique namespace.  It is RECOMMENDED that the last tuple of the
track namespace contain a broadcast timestamp to ensure uniqueness.

Within the namespace the publisher offers media tracks named `videoX` and
`audioX`, where X is an integer starting at 0 and increasing by 1 for each
additional track of a given type.

For example, if the publisher issues 2 audio tracks and 1 video track, the track
names available will be `video0`, `audio0`, and `audio1`.

The subscriber will consider all of those tracks belonging to the same
namespace as part of the same synchronization group (timestamps aligned to the
same timeline).

## Mapping Tracks to MoQT Object Model

For the video track, the publisher begins a new group at the start of each IDR
(so object 0 will be always an IDR Keyframe), and each group contains a single
subgroup.  Each object has the format described in {{object-format}}.

For the audio track, the publisher begins a new group with each audio object,
and each group contains a single subgroup.  Each object has the format described
in {{object-format}}.

TODO: Datagram forwarding preference could be used, but has problems if audio
frame does not fit in a single UDP payload.

## Timestamps

To avoid using fractional numbers and having to deal with rounding errors,
timestamps will be expressed with two integers:

* timestamp numerator (ex: PTS, DTS, duration)
* timebase

To convert a timestamp into seconds you just need to:
timestamp(s) = timestamp numerator / timebase

*Example:*

~~~
PTS = 11, timebase = 30

PTS(s) = 11/30 = 0.366666
~~~


## Object Format

All objects this protocol have the following format.

~~~
{
  Media Type (i)
  Media payload (..)
}
~~~
{: #media-object format title="MOQT Media object"}

* Media Type: Indicates what kind of media payload will follow.

|------|--------------------------------------|
| Code | Value                                |
|-----:|:-------------------------------------|
| 0x0  | Video H264 in AVCC with WCP          |
|------|--------------------------------------|
| 0x1  | Audio Opus bitstream                 |
|------|--------------------------------------|
{: #media-interop-meda-type title="Media Types"}

* Media payload: Media type specific payload

# Video H264 in AVCC with WCP Payload Format

~~~
{
  Seq ID (i)
  PTS Timestamp (i)
  DTS Timestamp (i)
  Timebase (i)
  Duration (i)
  Wall Clock (i)
  Metadata Size (i)
  Metadata (..)
  Payload (..)
}
~~~
{: #media-object-video-h264-avcc-wcp format title="MOQT Media video h264 WCP"}

* Seq ID: Monotonically increasing counter for this media track.

* PTS Timestamp: Presentation timestamp in timebase.

 TODO: Varint does NOT accept easily negative values, so it could be challenging
to encode at start (priming).

* DTS Timestamp: Display timestamp in timebase.  If B frames are not used, the
 encoder SHOULD set this to the same value as PTS.

TODO: Varint does NOT accept easily negative values, so it could be challenging
to encode at start (priming).

* Timebase: Denominator used to calculate `PTS`, `DTS`, and `Duration`.

* Duration: Duration of Payload in timebase.  It will be 0 if not set.

* Wall Clock: Epoch time in ms when this frame started being captured. It will
  be 0 if not set.

* Metadata Size: Size in bytes of the metadata section.  It will be 0 when no
  metadata is present.

* Metadata: Extra data needed to decode this stream. This will be
`AVCDecoderConfigurationRecord` as described in [ISO14496] section
5.3.3.1, with field `lengthSizeMinusOne` = 3 (So length = 4). If any other size
length is indicated (in `AVCDecoderConfigurationRecord`), the receiver SHOULD
close the session with a `Protocol Violation` error.
Any change in encoding parameters MUST send a new
`AVCDecoderConfigurationRecord`

* Payload: H264 with bitstream AVC1 format as described in [ISO14496]
section 5.3.  Using 4 bytes size field length.

# Audio Opus bitstream Payload Format

~~~
{
  Seq ID (i)
  PTS Timestamp (i)
  Timebase (i)
  Sample Freq (i)
  Num Channels (i)
  Duration (i)
  Wall Clock (i)
  Payload (..)
}
~~~
{: #media-object-audio-opus-WCP format title="MOQT Media audio Opus WCP"}

* Seq Id: Monotonically increasing counter for this media track.

* PTS Timestamp: Indicates PTS in timebase.

TODO: Varint does NOT accept easily negative, so it could be challenging to
encode at start (priming).

* Timebase: Denominator used to calculate `PTS` and `Duration`.

* Sample Freq: Sample frequency used in the original signal (before encoding).

* Num Channels: Number of channels in the original signal (before encoding).

* Duration: Duration of Payload in timebase.  It will be 0 if not set.

* Wall Clock: Epoch time in ms when this frame started being captured. It will
  be 0 if not set.

* Payload: Opus packets, as described in {{!RFC6716}} - section 3

# Conventions and Definitions

{::boilerplate bcp14-tagged}


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
