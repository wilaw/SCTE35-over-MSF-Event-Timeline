---
title: "SCTE35 transmission over MSF Event Timeline"
abbrev: "SCTE35 over MSF"
category: info

docname: draft-wilaw-moq-scte35-MSF-event-timeline
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
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
venue:
  group: "Media Over QUIC"
  type: "Working Group"
  mail: "moq@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/moq/"
  github: "wilaw/SCTE35-over-MSF-Event-Timeline"
  latest: "https://wilaw.github.io/SCTE35-over-MSF-Event-Timeline/draft-wilaw-moq-scte35-MSF-event-timeline.html"

author:
  - fullname: Will Law
    organization: Akamai
    email: "wilaw@akamai.com"
  - name: Suhas Nandakumar
    organization: Cisco
    email: snandaku@cisco.com

normative:
  MSF: I-D.draft-ietf-moq-msf-00
  JSON: RFC8259
  SCTE35:
    title: "SCTE 35: Digital Program Insertion Cueing Message"
    date: 2022
    target: https://www.scte.org/standards/library/catalog/scte-35-digital-program-insertion-cueing-message/
  SCTE214-1:
    title: "SCTE 214-1: MPEG DASH for IP-Based Cable Services Part 1 - MPD Constraints and Extensions"
    date: 2022
    target: https://www.scte.org/standards/library/catalog/scte-214-1-mpeg-dash-for-ip-based-cable-services-part-1-mpd-constraints-and-extensions/

informative:

...

--- abstract

Defines the transmission of SCTE35 data over MSF Event timeline tracks.


--- middle

# Introduction

MOQT Streaming Format {{MSF}} defines Event Timeline tracks as a generic mechanism for
transmitting ad hoc data associated with MSF media tracks. {{SCTE35}} markers signal ad insertion
points, program boundaries, and other broadcast events. This draft specifies how SCTE35
data can be transmitted using MSF Event Timeline tracks.

# Track properties
A MSF track carrying {{SCTE35}} or {{SCTE214-1}} data MUST

* declare a packaging value of "eventtimeline"
* declare an eventType value of "urn:scte:scte35:2022:bin" if using binary splice information
* declare an eventType value of "urn:scte:scte35:2022:xml" if using xml splice information

# SCTE 35 Mapping to MSF Event Timeline

## Record Structure

Each record in the MSF Event Timeline track for SCTE 35 MUST be a {{JSON}} object containing
an "m" field and a "data" field. The "data" field MUST be an object containing a scte35_payload
and a t (timing type) field.

The scte35_payload contains either the Base64-encoded binary representation of the
splice_info_section() or a string containing the escaped XML representation of the SCTE 35
message. The encoding format is determined by the eventType declared in the track properties.

~~~ json
{
  "m": 124500,
  "data": {
    "scte35_payload": "<scte35:SpliceInfoSection ... > ... </scte35:SpliceInfoSection>",
    "t": "pts"
  }
}

~~~

## Definition of the Media Time ("m")

The "m" field represents the media timeline offset in milliseconds. To ensure deterministic
behavior, "m" MUST be derived using the following precedence:

1. Timed Events (PTS): If a pts_time (binary) or ptsTime (XML) is present, "m" is the
   millisecond equivalent of the adjusted PTS:
   $m = \lfloor ((pts\_time + pts\_adjustment) \pmod{2^{33}}) / 90 \rfloor$
3. Immediate Events: If splice_immediate_flag is '1', "m" MUST be the media time of the
   first video frame following the message's insertion into the MSF stream.
4. Wallclock Events: If a utc_splice_time is present without a PTS, "m" MUST be the media
   time corresponding to that UTC moment.Other Events: For messages without timing
   (e.g., splice_event_cancel_indicator), "m" represents the media time at which the message
   is intended to be processed.

## The "t" Field (Timing Type)

The `t` field within the `data` object provides the receiver with context regarding the source
of the "m" value. This allows clients to distinguish between frame-accurate scheduled splices
and asynchronous "immediate" commands.

* `pts`: The "m" value was derived from a 90kHz PTS value.
* `immediate`: The "m" value was derived from the arrival time of an immediate flag.
* `utc`: The "m" value was derived from a UTC wallclock timestamp.
* `none`: The "m" value represents the emission time for a non-timed event.

## Payload Encoding Requirements

* Binary Payloads: When the track is configured for binary carriage, the `scte35_payload` MUST
  be a Base64-encoded string of the `splice_info_section()` as defined in {{SCTE35}}.
* XML Payloads: When the track is configured for XML carriage, the `scte35_payload` MUST be a
  string containing the XML representation as defined in {{SCTE35}}. Characters that are
  reserved in JSON (such as double quotes, backslashes, and control characters) MUST be properly
  escaped to maintain JSON validity as per {{JSON}}.

Implementations MUST NOT mix binary and XML payloads within the same MSF Media Timeline track
to ensure predictable parsing at the client.


# Examples

To illustrate the implementation of SCTE 35 within the MSF Event Timeline track, the following
examples demonstrate the mapping of various timing sources and the two supported encoding
formats (Binary and XML).

## Example 1: Binary Encoding with PTS Timing

This example shows the standard frame-accurate splice using a `pts_time`. The `m` value is the
calculated millisecond offset, and the payload is the Base64-encoded binary `splice_info_section()`.

~~~ json
[
  {
    "m": 480500,
    "data": {
      "scte35_payload": "/DAhAAAAAAAAAP/wFAUAAArXf+/+AAAAAH4AARSyAAAAAA==",
      "t": "pts"
    }
  },
  {
    "m": 510500,
    "data": {
      "scte35_payload": "/DAhAAAAAAAAAP/wFAUAAArYf+/+AAAAAH4AARSyAAAAAA==",
      "t": "pts"
    }
  }
]

~~~

## Example 2: XML Encoding with Immediate Flag

In this scenario, the track is configured for XML. The first record illustrates an "immediate" event
(no pre-calculated PTS), while the second shows a standard timed event. Note the JSON-escaped quotes
within the XML string.

~~~ json
[
  {
    "m": 62000,
    "data": {
      "scte35_payload": "<SpliceInfoSection><SpliceInsert spliceImmediateFlag=\"1\" eventId=\"101\"/></SpliceInfoSection>",
      "t": "immediate"
    }
  },
  {
    "m": 92000,
    "data": {
      "scte35_payload": "<SpliceInfoSection><TimeSignal><SpliceTime ptsTime=\"8280000\"/></TimeSignal></SpliceInfoSection>",
      "t": "pts"
    }
  }
]
~~~

## Example 3: Mixed Timing (UTC and Cancellation)

This example demonstrates the robustness of the "t" field, showing a message anchored to a
UTC wallclock and a subsequent "none" type record used for an event cancellation.

~~~ json
[
  {
    "m": 1500000,
    "data": {
      "scte35_payload": "/DApAAAAAAAAAP/wBQb+AAAAAAAfAh1zY3RlMzU6U2VnbWVudGF0aW9uRGVzY3JpcHRvcg==",
      "t": "utc"
    }
  },
  {
    "m": 1500500,
    "data": {
      "scte35_payload": "/DAWAAAAAAAAAP/wBQIAAAAAf3/yD77y",
      "t": "none"
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
| urn:scte:scte35:2022:bin       | SCTE-35 binary splice_info_section | this             |
| urn:scte:scte35:2022:xml       | SCTE-35 XML representation         | this             |



# Acknowledgments
The IETF moq workgroup.
