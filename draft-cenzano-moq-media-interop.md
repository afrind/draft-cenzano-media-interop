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

informative:


--- abstract

This protocol can be used to send and receive video and audio over Media over QUIC Transport [MOQT].

--- middle

# Introduction

This protocol specifies a simple mechanism for sending media (video and audio) over MOQT for both live-streaming and VC style use cases.  The protocol is flexible in order to support this range of use cases.

The following parameters can be updated in the middle of a the track (ex: frame rate, resolution, codec, etc)

The protocol defines a low overhead packager (not LoC [loq]), and is extensible
to other formats such as FMP4.

# Protocol Operation

## Track Names

The publisher selects a namespace of their choosing, and sends an ANNOUNCE
message for this namespace.

Within the publisher namespace there are two tracks with fixed names: `video`
and `audio`.

## Mapping Tracks to MoQT Object Model

For the video track, the publisher begins a new group at the start of each IDR
(so object 0 will be always an IDR Keyframe), and each group contains a single
subgroup.  Each object has the format described in {{video-object-format}}.

For the audio track, the publisher begins a new group with each audio object,
and each group contains a single subgroup (OR we could align the group numbers
with video and make a subgroup per object.  Each object has the format described
in {{audio-object-format}}.

TODO: Datagram forwarding preference could be used, but has problems if audio
frame does not fit in a single UDP payload.

## Object Format

### Video Object Format

~~~
{
  Media Type (i)
  Seq ID (i)
  PTS Timestamp (i)
  DTS Timestamp (i)
  Timebase (i)
  Duration (i)
  Wallclock (i)
  Metadata Size (i)
  Metadata (..)
  Payload (..)
}
~~~


Media Type

Specifies the meaning of all the data sent after this field
It keeps this packager extensible to any other options (such as: fmp4, etc)
It can be:
- VideoLOCH264AVCC = 0


Seq ID

Monotonically increasing counter for this media track


PTS Timestamp

Indicates PTS in timebase

TODO: Varint does NOT accept easily negative, so it could be challenging to
encode at start (priming)


DTS Timestamp

Not needed if B frames are NOT used, in that case should be same value as PTS

TODO: Varint does NOT accept easily negative, so it could be challenging to
encode at start (priming)


Timebase

Units used in PTS, DTS, and duration


Duration

Duration in timebase
It will be 0 if not set


Wall Clock

EPOCH time in ms when this frame started being captured
It will be 0 if not set


Metadata Size

Size in bytes of the metadata section
It can be 0 if no metadata is sent


Metadata

Extradata needed to decode this stream
For `mediaType == VideoLOCH264AVCC` this field will be 
AVCDecoderConfigurationRecord as described in ISO/IEC 14496-15 section 5.3.3.1,
with field `lengthSizeMinusOne` = 3 (So length = 4). If any other size length is
indicated (in AVCDecoderConfigurationRecord) we should error with “Protocol
violation”


Payload

H264 with bitstream AVC1 format as described in iso14496-15 section 5.3.
Using 4bytes size field length.
If any other size length is indicated (in AVCDecoderConfigurationRecord) we
should error with “Protocol violation”.
Any change in encoding parameters MUST send a new AVCDecoderConfigurationRecord
in Metadata


## Audio Object Format

~~~
{
  Media Type (i)
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


Media Type

Specifies the meaning of all the data sent after this field
It keeps this packager extensible to any other options (such as AAC-ASC, etc)
It can be:
- AudioLOCOpus = 1


Seq Id

Monotonically increasing counter for this media track

PTS Timestamp

Indicates PTS in timebase

TODO: Varint does NOT accept easily negative, so it could be challenging to
encode at start (priming)


Timebase

Units used in PTS, DTS, and duration

Sample Freq

Sample frequency used in the original signal (before encoding)


Num Channels

Number of channels in the original signal (before encoding)


Duration

Duration in timebase
It will be 0 if not set


Wallclock

EPOCH time in ms when this frame started being captured
It will be 0 if not set


Payload

Opus packets, as described in rfc6716 - section 3


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
