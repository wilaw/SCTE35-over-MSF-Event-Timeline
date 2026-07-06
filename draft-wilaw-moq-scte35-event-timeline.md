---
title: "SCTE35 transmission over MSF Event Timeline"
abbrev: "SCTE35 over MSF"
category: info

docname: draft-wilaw-moq-scte35-event-timeline-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Media Over QUIC"
keyword:
 - MSF
 - Eventtimeline
 - SCTE35
 - MOQT
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "wilaw/SCTE35-over-MSF-Event-Timeline"
  latest: "https://wilaw.github.io/SCTE35-over-MSF-Event-Timeline/draft-wilaw-moq-scte35-event-timeline.html"

author:
  - fullname: Will Law
    organization: Akamai
    email: "wilaw@akamai.com"
  - name: Suhas Nandakumar
    organization: Cisco
    email: snandaku@cisco.com

normative:
  MSF: I-D.draft-ietf-moq-msf-01
  JSON: RFC8259
  SCTE35:
    title: "SCTE 35: Digital Program Insertion Cueing Message"
    date: 2022
    target: https://www.scte.org/standards/library/catalog/scte-35-digital-program-insertion-cueing-message/

informative:

...

--- abstract

Defines the transmission of SCTE 35 data over MSF Event Timeline tracks.


--- middle

# Introduction

MOQT Streaming Format {{MSF}} defines Event Timeline tracks as a generic mechanism for
transmitting ad hoc data associated with MSF media tracks. {{SCTE35}} markers signal ad insertion
points, program boundaries, and other broadcast events. This draft specifies how SCTE35
data can be transmitted using MSF Event Timeline tracks.

# Track properties
An MSF track carrying {{SCTE35}} data MUST

* declare a packaging value of "eventtimeline"
* declare an eventType value of "urn:scte:scte35:2022:bin" if using binary splice information
* declare an eventType value of "urn:scte:scte35:2022:xml" if using xml splice information

# SCTE 35 Mapping to MSF Event Timeline

## Record Structure

Each record in the MSF Event Timeline track for SCTE 35 MUST be a JSON object conforming to
the Event Timeline data format defined in [MSF] Section 8.1: it MUST contain exactly one
index reference field (t, l, or m), selected per the precedence rules in Section 3.2, and a
data field.

The data field MUST be an object containing a single scte35_payload member. The scte35_payload
contains either the Base64-encoded binary representation of the splice_info_section() or a
string containing the escaped XML representation of the SCTE 35 message, per the eventType
declared in the track properties (Section 2).

~~~ json
{
  "m": 480500,
  "data": {
    "scte35_payload": "/DAhAAAAAAAAAP/wFAUAAArXf+/+AAAAAH4AARSyAAAAAA=="
  }
}
~~~

~~~ json
{
  "t": 1756885678361,
  "data": {
    "scte35_payload": "/DApAAAAAAAAAP/wBQb+AAAAAAAfAh1zY3RlMzU6U2VnbWVudGF0aW9uRGVzY3JpcHRvcg=="
  }
}
~~~

~~~ json
{
  "l": [42, 0],
  "data": {
    "scte35_payload": "<SpliceInfoSection><SpliceInsert spliceImmediateFlag=\"1\" eventId=\"101\"/></SpliceInfoSection>"
  }
}
~~~

Since the index reference field itself identifies the timing source used for a record, no
additional field is required inside data to disambiguate it. Receivers requiring finer-grained
classification (e.g., distinguishing an immediate splice from an event cancellation) MAY inspect
the scte35_payload itself, since this information is already present in the SCTE 35 message
(splice_immediate_flag, splice_event_cancel_indicator, etc.) and duplicating it in the envelope
would be redundant.

## Index Selection

Each record MUST select its index reference field according to the following precedence, applied
to the SCTE 35 message being carried:

* PTS-timed events: If a pts_time (binary) or ptsTime (XML) is present, the record MUST use m,
  the media time in milliseconds, computed as: floor(((pts_time + pts_adjustment) mod 2^33) / 90).
  This value MUST be expressed in the same coordinate space as the media time defined for the
  corresponding Media Timeline track or template ([MSF] Section 7.1.1) of the track(s) named in
  this track's depends attribute — that is, pts_time and the referenced media track's timestamps
  MUST share a common zero-point and epoch. Publishers MUST resolve any PTS discontinuities or
  2^33 wraparound in the source SCTE 35 stream before computing m, so that the resulting value
  remains monotonic and consistent with the associated media track's timeline.
* Wallclock-timed events: If a utc_splice_time is present without a pts_time, the record MUST
  use t, set directly to the UTC time expressed as milliseconds since the Unix epoch, per
  [MSF] Section 8.1. No conversion to media time is required or permitted; this avoids requiring
  the publisher to maintain a UTC-to-media-time mapping solely for signaling purposes.
* Immediate events: If splice_immediate_flag is 1 and no pts_time is present, the record MUST
  use l, set to the MOQT Location — [Group ID, Object ID] — of the media Object at or immediately
  preceding which the splice is to take effect. This anchors the event to an actual encoded Object
  in the dependent media track rather than an approximated wallclock or media time.
* Other events: For messages that carry no timing information of their own
  (e.g., splice_event_cancel_indicator, splice_null()), the record MUST use l, set to the MOQT
  Location of the media Object with which the record is associated at the time of publication.

Publishers MUST use only one index reference field per record, per [MSF] Section 8.1. Because the
four cases above use t, m, and l at different times within the same track, this track is a case
where index reference types intentionally vary record-to-record, rather than following the
"SHOULD use the same index reference type" guidance in [MSF] Section 8.1 — that guidance is best
suited to tracks with a single, uniform timing source, which SCTE 35 signaling is not.

## Payload Encoding Requirements

* Binary Payloads: When the track is configured for binary carriage, the scte35_payload MUST be
  a Base64-encoded string of the splice_info_section() as defined in [SCTE35].
* XML Payloads: When the track is configured for XML carriage, the scte35_payload MUST be a string
  containing the XML representation as defined in [SCTE35]. Characters that are reserved in JSON
  (such as double quotes, backslashes, and control characters) MUST be properly escaped to maintain
  JSON validity as per [JSON].

Implementations MUST NOT mix binary and XML payloads within the same MSF Event Timeline track
to ensure predictable parsing at the client.

# Examples

To illustrate the implementation of SCTE 35 within the MSF Event Timeline track, the following
examples demonstrate the mapping of various timing sources — using MSF's native t, l, and m index
reference fields — and the two supported encoding formats (Binary and XML).

## Example 1: Binary Encoding with PTS Timing

This example shows the standard frame-accurate splice using a pts_time. Per Section 3.2 case 1,
the record uses m, the calculated millisecond offset, and the payload is the Base64-encoded binary
splice_info_section().

~~~ json
[
  {
    "m": 480500,
    "data": {
      "scte35_payload": "/DAhAAAAAAAAAP/wFAUAAArXf+/+AAAAAH4AARSyAAAAAA=="
    }
  },
  {
    "m": 510500,
    "data": {
      "scte35_payload": "/DAhAAAAAAAAAP/wFAUAAArYf+/+AAAAAH4AARSyAAAAAA=="
    }
  }
]
~~~

## Example 2: XML Encoding with Immediate Flag

In this scenario, the track is configured for XML. The first record illustrates an "immediate" event
(splice_immediate_flag="1", no PTS) — per Section 3.2 case 3, it uses l, the MOQT Location of the
media Object at which the splice takes effect. The second record shows a standard PTS-timed event,
using m. Note the JSON-escaped quotes within the XML string.

~~~ json
[
  {
    "l": [15, 0],
    "data": {
      "scte35_payload": "<SpliceInfoSection><SpliceInsert spliceImmediateFlag=\"1\" eventId=\"101\"/></SpliceInfoSection>"
    }
  },
  {
    "m": 92000,
    "data": {
      "scte35_payload": "<SpliceInfoSection><TimeSignal><SpliceTime ptsTime=\"8280000\"/></TimeSignal></SpliceInfoSection>"
    }
  }
]
~~~

## Example 3: Mixed Timing (UTC and Cancellation)

This example demonstrates a track that varies its index reference field record-to-record, as permitted
by Section 3.2. The first record is anchored to a UTC wallclock moment — per case 2, it uses t directly,
with no media-time conversion required. The second is a splice_event_cancel_indicator cancellation with
no timing information of its own — per case 4, it uses l, referencing the Location of the media Object
with which it is associated at the time of publication.

~~~ json
[
  {
    "t": 1756885678361,
    "data": {
      "scte35_payload": "/DApAAAAAAAAAP/wBQb+AAAAAAAfAh1zY3RlMzU6U2VnbWVudGF0aW9uRGVzY3JpcHRvcg=="
    }
  },
  {
    "l": [15, 3],
    "data": {
      "scte35_payload": "/DAWAAAAAAAAAP/wBQIAAAAAf3/yD77y"
    }
  }
]
~~~


# Security Considerations

The carriage of SCTE 35 signals within the MSF Event Timeline track inherits the security
considerations of both the underlying MSF transport and the SCTE 35 standard itself.

## Integrity and Authenticity
SCTE 35 messages are frequently used to trigger high-value business logic, such as the
insertion of advertising or the enforcement of viewing blackouts. If the MSF track is not
protected by transport-layer security (e.g., TLS) or object-level signing, an attacker could
modify the "m" timing value or the "scte35_payload" to disrupt ad delivery or cause
unauthorized content transitions.

## Payload Validation
Implementations MUST treat the scte35_payload as untrusted data. Receivers should implement
robust parsing for both Base64-encoded binary data and XML strings to prevent buffer overflow
attacks or XML External Entity (XXE) exploits. Specifically, when parsing XML-encoded SCTE 35,
parsers SHOULD be configured to disallow DTDs and external entities. Large scte35_payload
strings (especially in XML) could lead to memory exhaustion if not bound-checked by the
JSON parser.

## Denial of Service (DoS)
An attacker could inject a high frequency of timeline records with conflicting "m" values.
This could lead to "event thrashing," where the client device is forced to rapidly switch states,
potentially leading to resource exhaustion or a degraded user experience.


# IANA Considerations

This document adds two entries to the "MSF Event Timeline Types" registry.

| Event Type                     | Description                        | Specification    |
|:===============================|:===================================|:=================|
| urn:scte:scte35:2022:bin       | SCTE 35 binary splice_info_section | this             |
| urn:scte:scte35:2022:xml       | SCTE 35 XML representation         | this             |


# Acknowledgments
The IETF moq workgroup.
