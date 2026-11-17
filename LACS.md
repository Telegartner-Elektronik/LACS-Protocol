# Lift Alarm Communication Standard (LACS)

**Version:** 1.0.0-rc1

**Last Updated:** 2026-11-17

**Maintainer:** Telegärtner Elektronik GmbH

**Canonical Source Repository:**
<https://codeberg.org/telegaertner-elektronik/LACS-Protocol>

**Contributors:**

- IOT Communications International AB
- Serv24 GmbH

## Status - Release Candidate

> [!NOTE]
>
> This document is a **release candidate** of the **Lift Alarm Communication Standard (LACS)** 1.0.0.
> It is open for public review until 2026-12-04. Please send your feedback via the canonical source repository.
> Changes until the final version 1.0.0 are limited to corrections that result from the review.

## Licence

This documentation is published under the **Creative Commons Attribution-ShareAlike (CC BY-SA)** licence.
You are free to use, copy, distribute, and modify this work, provided that proper attribution is given and that any derived works are released under the same licence.

We explicitly encourage **open collaboration and feedback**. If you identify inconsistencies, security concerns, or technical improvements, or if you would like to propose extensions to the current specification, please share your suggestions openly.
The aim of LACS is to evolve transparently and to serve as an open, widely applicable communication standard for lift alarm systems.

### Additional Permissions for Standardisation Bodies

In addition to the rights granted under the Creative Commons Attribution-ShareAlike 4.0 International licence (CC BY-SA 4.0), the authors grant CEN, CENELEC, ETSI and other recognised standardisation bodies a non-exclusive, irrevocable permission to reproduce, adapt, incorporate and publish this specification or parts of it in any formal standard, without the obligation to apply the Share-Alike requirement of CC BY-SA to the resulting standard or its derivatives.

This permission applies exclusively to the creation of formal standards and does not limit the CC BY-SA licensing that applies to all other users.

## Purpose

The purpose of this document is to standardise the practical implementation of lift alarm in accordance with prEN 81-28:2025 on the basis of CLC/TS 50134-9:2018. The Information Note "Voice over IP for EN 81-28 communication system" issued by the ELA TELCO WG in January 2024 is taken into account.

It also takes into account optional practical applications that go beyond the basic requirements of EN 81-28.

The supplementary specifications in this document are intended to achieve a protocol that can be implemented as universally as possible and supported by as many manufacturers and alarm receiving centres as possible.

The document also contains specific examples to facilitate implementation and avoid misunderstandings. Together with CLC/TS 50134-9 it contains everything needed to implement LACS.

A LUC MAY support further alarm paths of other types in addition to LACS, e.g. a conventional dial-up path. This specification describes the LACS alarm path only. A LUC SHOULD provide one or more fallback alarm paths for the case that LACS is not available.

### Motivation

The motivation for creating this specification arises from several gaps and insufficiently specified behaviours found in the ELA Information Note and in CLC/TS 50134-9:

- Missing precise definitions regarding specific processes, message properties, and when particular codes are to be used.
  Without such clarification, independent implementations would differ in critical aspects, resulting in derivatives or dialects instead of a universally applicable standard.
- Lack of a clearly defined call sequence that ensures conformity with both normative and application-specific requirements for lift alarms
- No consistent session association for all messages exchanged from the initial alarm trigger to the end of the alarm
- Insufficient coverage of relevant message types already essential in state of the art lift alarm systems, including:
  - Alarm button failure
  - Microphone/speaker failure
  - Internal bus failure
  - Communication voice failure
- No mechanism for validating bidirectional communication in the voice channel, which was previously ensured implicitly by DTMF-based protocols
- No provision for extended, asynchronous communication between LUC and ARC, necessary for features commonly implemented in modern lift alarm systems such as:
  - Text-based alarm communication
  - Camera integration and image retrieval
- No method for the LUC to announce its supported extended capabilities to the ARC (e.g., TBUI features, camera functions, ...)

## Document Structure

LACS uses two SIP MESSAGE content types:

- **`application/scaip+xml`** covers the lift alarm communication as defined by TS 50134-9: all requests of the LUC towards the ARC, i.e. voice alarms, test transmissions, status events and failure notifications. LACS extensions to `scaip+xml` are intended to be proposed for adoption into TS 50134-9 through ELA and CENELEC.
- **`application/lacs+xml`** carries the commands of the ARC to the LUC (`mty=CO`), e.g. end of alarm triggered by the ARC, callback and voice channel test, and the optional text-based alarm communication (TBUI) and camera functions. It goes beyond the scope of TS 50134-9 and is specific to LACS.

The LUC and the ARC MUST support both content types (see [Conformance](#conformance)).

The specification is split across the following documents:

| Document                               | Scope                                                                                     |
| -------------------------------------- | ----------------------------------------------------------------------------------------- |
| [LACS.md](LACS.md)                     | Overview, terms, architecture, configuration and timers, SIP profile, security, conformance |
| [LACS-scaip-xml.md](LACS-scaip-xml.md) | `application/scaip+xml` — message properties for EN 81-28 alarm communication             |
| [LACS-lacs-xml.md](LACS-lacs-xml.md)   | `application/lacs+xml` — commands of the ARC, TBUI and camera                             |
| [LACS-usecases.md](LACS-usecases.md)   | Procedures with rules, sequence diagrams and examples                                     |
| [xml-schema/](xml-schema/)             | XML schemas for `scaip+xml` and `lacs+xml`                                                |
| [FAQ.md](FAQ.md)                       | Frequently asked questions on design decisions (informative)                              |
| [GOVERNANCE.md](GOVERNANCE.md)         | Use of the name LACS, maintainer                                                          |
| [CHANGELOG.md](CHANGELOG.md)           | Changes per version; breaking changes are marked                                          |

## Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in BCP 14 [RFC 2119] [RFC 8174] when, and only when, they appear in all capitals, as shown here. This applies to all documents of LACS.

Sequence diagrams and examples are informative. Normative requirements are given in the text and in the numbered rules.

The examples use the values of one sample installation, taken from ranges reserved for documentation or, for telephone numbers, for drama use:

| Item | Value in the examples |
| ---- | --------------------- |
| Domain of the ARC | `arc.example.com`, alternative ARC `arc2.example.com` (RFC 2606) |
| SIP URIs of the ARC | `sip:reception@arc.example.com` (`alarm_uri`); `sips:operator17@arc.example.com`, `sips:service@arc.example.com`, `sips:echo@arc.example.com` (`tnu`); alternative ARC `sips:operator5@arc2.example.com` |
| SIP URI of the LUC | `sip:14100012345@arc.example.com` |
| IP addresses | LUC `192.0.2.10`, ARC `198.51.100.20`, alternative ARC `203.0.113.30` (RFC 5737); IPv6 `2001:db8::10` (RFC 3849) |
| Ports | SIP over TLS `5061`, RTP `40000` and `40002` |
| `cid` | `14100012345` |
| `did` | one local unit in the car, `did` `1`; `iba` is therefore omitted |
| Installation with TBUI and camera | examples of _Text-based Alarm Communication_ and _Camera support_: second controller `cid` `14100067890`, one local unit `did` `1` with TBUI and camera (`cap` `0011,0012,0013,0014,0021`) |
| `asid` | one fixed UUID per session and use case |
| `ref` | one value per request; the `mrs` repeats the value of its `mrq` |
| Nameplate of controller and local unit | `Example Lift Phones`, `ELP-GW 200 rev C`, `3.4.1`, IMEI `356938035643809`; local unit `ELP-CAR 10 rev B`, `2.1.0` |
| MSISDN of the SIM card (`npcms`) | `+447700900123` (Ofcom number for drama use; E.164 has no range reserved for documentation) |

Placeholders in braces, e.g. `{controller_id}`, describe a format in rules and appear in examples only for content that does not matter for understanding, e.g. image data.

## Normative References

The following normative documents contain provisions which, through reference in this text, constitute provisions of this specification. For dated references, only the edition cited applies. For undated references, the latest edition of the referenced document (including any amendments) applies.

- **prEN 81-28:2025** (draft) – Safety rules for the construction and installation of lifts – Lifts for the transport of persons and goods – Part 28: Two-way communication system. This specification follows the draft and will be updated to the published edition of EN 81-28.
- **CLC/TS 50134-9:2018** – Alarm systems – Social alarm systems – Part 9: Internet Protocol (IP) based transmission using SCAIP (Social Care Alarm Internet Protocol)
- **RFC 2119**, **RFC 8174** – Key words for use in RFCs to Indicate Requirement Levels (BCP 14)
- **RFC 3261** – SIP: Session Initiation Protocol
- **RFC 3311** – The Session Initiation Protocol (SIP) UPDATE Method
- **RFC 3326** – The Reason Header Field for the Session Initiation Protocol (SIP)
- **RFC 3428** – Session Initiation Protocol (SIP) Extension for Instant Messaging
- **RFC 4103** – RTP Payload for Text Conversation
- **RFC 7616** – HTTP Digest Access Authentication
- **RFC 3665** – Session Initiation Protocol (SIP) Basic Call Flow Examples
- **RFC 4566** – SDP: Session Description Protocol
- **RFC 5626** – Managing Client-Initiated Connections in the Session Initiation Protocol (SIP Outbound)
- **RFC 5922** – Domain Certificates in the Session Initiation Protocol (SIP)
- **RFC 4961** – Symmetric RTP / RTP Control Protocol (RTCP)
- **RFC 3550** – RTP: A Transport Protocol for Real-Time Applications
- **RFC 3711** – Secure Real-time Transport Protocol (SRTP)
- **RFC 4568** – Session Description Protocol (SDP) Security Descriptions for Media Streams
- **RFC 4733** – RTP Payload for DTMF Digits, Telephony Tones, and Telephony Signals
- **RFC 9562** – Universally Unique IDentifiers (UUIDs)
- **RFC 3339** – Date and Time on the Internet: Timestamps
- **RFC 7714** – AES-GCM Authenticated Encryption in the Secure Real-time Transport Protocol (SRTP)
- **RFC 5246** – The Transport Layer Security (TLS) Protocol Version 1.2
- **ITU-T G.711** – Pulse code modulation (PCM) of voice frequencies
- **ITU-T G.729** – Coding of speech at 8 kbit/s
- **RFC 5280** – Internet X.509 Public Key Infrastructure Certificate and Certificate Revocation List (CRL) Profile
- **RFC 8446** – The Transport Layer Security (TLS) Protocol Version 1.3
- **ISO 639-1** – Codes for the representation of names of languages – Part 1: Alpha-2 code

## Informative References

- **ELA – Information Note (January 2024)** – _Voice over IP for EN 81-28 communication systems_. Its relevant content has been taken into account in this specification.
- **ELA – Information Note (2025)** – _Cybersecurity and operational aspects of VoIP two-way communication systems (based on EN81-28:2025)_. Its relevant content has been taken into account in this specification.
- **EN 50136-1** – Alarm systems – Alarm transmission systems and equipment – Part 1: General requirements for alarm transmission systems
- **EN 50136-2** – Alarm systems – Alarm transmission systems and equipment – Part 2: Requirements for Supervised Premises Transceiver (SPT)
- **EN 50136-3** – Alarm systems – Alarm transmission systems and equipment – Part 3: Requirements for Receiving Centre Transceiver (RCT)
- **CLC/TS 50136-9** – Alarm systems – Alarm transmission systems and equipment – Part 9: Requirements for common protocol for alarm transmission using the Internet Protocol (IP)
- **ETSI TS 103 479** – Emergency communications (EMTEL); Requirements for Internet Protocol-based emergency alarm systems
- **ISO 8102-20** – Electrical requirements for lifts, escalators and moving walks – Part 20: Cybersecurity
- **IEC 62443-4-2** – Security for industrial automation and control systems – Part 4-2: Technical security requirements for IACS components
- **IEEE 802.1AR** – Secure Device Identity
- **RFC 7030** – Enrollment over Secure Transport (EST)
- **BetrSichV / TRBS 3121** – German Ordinance on Industrial Safety and Health and Technical Rule for Operational Safety 3121 (operation of lifts, including the lift attendant)
- **SOSAB P100 Protocol** – User Controlled Reporting (events V001 to V010)

LACS is not an alarm transmission system according to EN 50136 and does not claim conformity with it. Individual mechanisms follow EN 50136 as state of the art: the event time (`etm`), the detection of repeated requests, the retention of single events until they are answered, randomised retry times and the supervision of the alarm path by the registration.

## Terms, definitions and abbreviations

In addition to the definitions from TS 50134-9, this section defines the specific aspects in the context of lift alarm systems. This specification deliberately uses the terms alarm and Alarm Receiving Centre (ARC); prEN 81-28:2025 speaks of the alert and the reception equipment.

### LUC

Local Unit and Controller: the lift alarm equipment on the lift side, consisting of a controller and one or more local units.

### LACS

Lift alarm transmission protocol based on SIP + TS 50134-9 + Information Note from ELA WG TELCO with practical extensions for lift alarm, shared under a CC BY-SA licence.

### Controller

Refers to the transmission device (gateway) with which data and voice can be transmitted via SIP + TS 50134-9. A single controller can serve multiple local units and thus multiple lift cars (e.g., in groups with several lifts next to each other). The controller is identified by its `controller-id` (`cid`).

### Internal Bus

Internal communication interface via which one or more local units are connected to the controller and exchange data and voice. This can be an analogue telephone connection or a digital protocol such as IP. A controller has exactly one internal bus; each local unit on it has an internal bus address (`iba`).

### Local Unit

Refers to the alarm telephone. Each local unit has one or more alarm buttons and one or more voice units connected to it.

Several local units connected to one controller can be installed at different locations in the same lift (e.g. in the cabin, on the cabin roof or in the lift shaft) or be installed on different lifts (e.g. in groups with several lifts next to each other).

### Device Component

Components belonging to a local unit, such as:

- Alarm button
- Voice unit (microphone and speaker)
- Camera
- User interface for text-based communication (TBUI)

### Voice Unit

A microphone and a speaker of a local unit at one location, e.g. in the car, on the car roof or in the pit. A local unit has up to five voice units. Each voice unit forms a unit with the alarm button at the same location; both are identified by the device components `002` to `006` (`dco`, "Alarm Button / Voice Unit 1" to "5").

### Service Voice Call

A voice call requested by the ARC outside an alarm (`aid=0223`) and placed by the LUC, in which the audio settings of the voice units of a local unit can be changed permanently. An alarm always takes precedence (see [Service voice call](LACS-usecases.md#service-voice-call)).

### TBUI

Text-based user interface that enables text alarm communication between a trapped passenger and the Alarm Receiving Centre (ARC). This text communication is a supplement to standard audio communication via voice.

A TBUI can be integrated into the Local Unit or implemented through a separate user interface component connected via the internal bus. Each TBUI MUST be associated with exactly one Local Unit, and all TBUI-related messages MUST be transmitted within the same session context as the corresponding voice alarm. The TBUI provides predefined questions and answers, free-text input (if supported), dynamic answer options and language selection.

### Alarm Path

A way over which the LUC transmits an alarm to an ARC, e.g. LACS or a conventional dial-up path. This specification describes the LACS alarm path only.

### Session

All messages and calls that belong to one process: a lift alarm from the first voice request to the end of alarm, a voice channel test, a callback or a service voice call requested by the ARC, or a test transmission requested by the ARC. A session is identified by its alert session ID (`asid`, see [alert-session-id (asid)](LACS-scaip-xml.md#alert-session-id-asid)).

### Single Event

A message of the LUC that does not belong to a session with several messages, e.g. a test transmission, a power failure or a failure report. Its `asid` is valid for one request and its response only.

### Acknowledgement

Confirmation by the ARC that it has received the alarm (prEN 81-28:2025 §4.2.4). In LACS the ARC acknowledges the alarm by answering the voice request with `snu=0`; the voice call follows (see [Acknowledgement according to EN 81-28](LACS-usecases.md#acknowledgement-according-to-en-81-28)).

### AMS

Alarm Management System: the software of the ARC that processes alarms and messages and presents them to the operators.

### EOA

End of alarm (`stc=0201`): the LUC reports that the alarm has ended, after it has been reset at the local unit or by the ARC (`aid=0212`).

## Architecture

```mermaid
graph LR

    subgraph LUC["LUC"]
        DC1@{ shape: processes, label: "Device Component" }
        DC2@{ shape: processes, label: "Device Component" }
        DC3@{ shape: processes, label: "Device Component" }

        LU1["Local Unit 1"]
        LU2["Local Unit 2"]
        LU3["Local Unit 3"]
        BUS@{ shape: f-circ, label: "Internal Bus" }
        CTR["Controller"]
        ES["Emergency power supply"]
    end
    LUC["LUC"]
    ARC["ARC"]

    %% Connections
    DC1 --- LU1
    DC2 --- LU2
    DC3 --- LU3
    LU1 --- BUS
    LU2 --- BUS
    LU3 --- BUS
    BUS -- "Internal Bus" --- CTR
    CTR -- "LACS + SRTP" --> ARC
```

Each controller is identified by its controller-id (`cid`).

Each local unit is identified by `cid` and its device-id (`did`). If the controller has more than one local unit, the internal bus address (`iba`) is transmitted in addition.

## Basic Concepts

### Protocol Layer Model

This overview shows the communication channels and the underlying layers:

| Purpose              | SIP Signalling                                                           | SCAIP XML                                        | Audio for voice alarms                                  | LACS XML                                        | **JPEG Snapshot**<br />Optional                        | **Multimedia Services** _(RTT + Video, ...)_<br />Optional    |
| -------------------- | ------------------------------------------------------------------------ | ------------------------------------------------ | ------------------------------------------------------- | ----------------------------------------------- | ------------------------------------------------------ | ------------------------------------------------------------- |
| **7 – Application**  | SIP, SDP                                                                 | SIP MESSAGE with `application/`<br />`scaip+xml` | RTP payloads: G.711 A-law (mandatory), G.729 (mandatory for the ARC)    | SIP MESSAGE with `application/`<br />`lacs+xml` | multipart MIME (`application/`<br />`lacs+xml` + JPEG) | T.140 (RFC 4103), RTP video payloads (H.264/H.265 or similar) |
| **6 – Presentation** | TLS ≥ 1.2 with mutual authentication (X.509); optionally SIP Digest      | TLS ≥ 1.2                                        | SRTP (AES-128 minimum)                                  | TLS ≥ 1.2                                       | JPEG base64 inside multipart                           | SRTP (AES-128 minimum)                                        |
| **5 – Session**      | SIP dialog layer, REGISTER                                               | SCAIP model (`ref`, `asid`)                      | RTP/SDP session                                         | LACS model (`ref`, `asid`)                      | snapshot request session (`ref`)                       | RTP/SDP session                                               |
| **4 – Transport**    | TCP with TLS (mandatory)                                                 | SIP MESSAGE transport                            | UDP (RTP/RTCP)                                          | SIP MESSAGE transport                           | SIP MESSAGE transport                                  | UDP (RTP/RTCP)                                                |
| **3 – Network**      | IPv4 / IPv6                                                              | IPv4 / IPv6                                      | IPv4 / IPv6                                             | IPv4 / IPv6                                     | IPv4 / IPv6                                            | IPv4 / IPv6                                                   |

### Versioning

The version of LACS has the form `MAJOR.MINOR.PATCH` (header of this document), optionally with the suffix `-rc<n>` for a release candidate:

- **PATCH:** editorial corrections and clarifications without any change on the wire or in the behaviour of LUC and ARC.
- **MINOR:** backwards compatible additions within the same major version (TS 50134-9 §7.2), e.g. optional elements, codes and functions. A receiver ignores elements and values it does not know.
- **MAJOR:** incompatible changes.

Every `mrq` and `mrs` carries the protocol name and `MAJOR.MINOR` of the sender in `pve`, e.g. `LACS/1.0`; PATCH and suffix are not transmitted. All release candidates of a version therefore send the same `pve` as the final version. Functions marked as experimental are excluded from the compatibility rules and can change in a later minor version. For the rules see [protocol-version (pve)](LACS-scaip-xml.md#protocol-version-pve).

### LUC Configuration

Settings and information configured at the Local Unit and Controller (LUC)

| Name                       | Type              | Default        | Description |
| -------------------------- | ----------------- | -------------- | ----------- |
| `controller_id`            | String            |                | `cid`, 1 to 32 digits |
| `device_id_list`           | String list       |                | Device ID (`did`) of each local unit, each with its internal bus address (`iba`) if the controller has more than one local unit |
| `sip_domain`               | String            |                | Host part of the SIP URIs (e.g. in the `From` and `To` headers) |
| `sip_outbound_proxy`       | String            | `sip_domain`   | IP address or DNS name. If the DNS name resolves to several addresses, the LUC uses them one after the other (see [SIP Profile](#sip-profile)) |
| `sip_port`                 | Integer           | 5061           | SIP port |
| `ca_certificate`           | X.509 certificate |                | CA certificate for the validation of the server certificate |
| `client_certificate`       | X.509 certificate |                | Operational client certificate of the controller, subject CN = `controller_id` (see [Authentication and Credentials](#authentication-and-credentials)) |
| `client_private_key`       | Private key       |                | Private key of the client certificate; unique per device, MUST NOT leave the device |
| `sip_username`             | String            |                | Optional - only if SIP Digest authentication is used in addition |
| `sip_password`             | String            |                | Optional - only if SIP Digest authentication is used in addition; unique per device |
| `alarm_uri`                | String            |                | Target SIP URI for `scaip+xml` and `lacs+xml` messages |
| `automatic_test_uri`      | String            | `alarm_uri`    | Optional - target SIP URI for test transmissions (`stc` 0102, 0103) |
| `automatic_test_interval` | Integer (h)       | 72             | Interval of the periodic tests and the test transmission, at most 72 h. prEN 81-28:2025 §4.2.8 requires an automatic test at least every 3 days. |
| `alarm_accept_timeout`     | Integer (s)       | 180            | Maximum time a voice alarm stays in HOLD (`snu=4`) on a given server before the LUC resets it there and tries the next server address (180 s = lower bound of TS 50134-9 §5.2). Shorter values are permitted for faster failover to the next server address; this deliberately deviates from TS 50134-9 §5.2. At most 240 s, so that the LUC turns to the next configured target (a further server address or the fallback alarm path) within 5 min (prEN 81-28:2025 §4.2.4). |
| `call_protection`          | Boolean           | `false`        | If `true`, the LUC accepts incoming calls only during an active session and only over the TLS connection of its registration (see [Call Protection](#call-protection)) |

### ARC Configuration

Settings and information configured in the ARC or its Alarm Management System

- The controller with its `cid` and each local unit with its `did`, for a lift alarm via LACS.
- The internal bus address (`iba`) of each device-id, if the controller has more than one local unit (default `1`).
- Expected interval of the test transmission in hours
- Exact address / location of the object
- Information and contact details needed to initiate and coordinate the rescue of passengers
- Information and contact details to initiate service work

The ARC does not need to know in advance which device type and version is installed and which extended functions it supports. The LUC reports manufacturer, type, firmware and hardware identifier in the nameplate elements (see [nameplate (np…)](LACS-scaip-xml.md#nameplate-np)) and its extended functions in `cap`.

### Timers

All times of LACS at a glance. Values marked _fixed_ are not configurable.

| Time | Side | Value | Specified in |
| ---- | ---- | ----- | ------------ |
| Uniqueness of `ref`, detection of repeated requests | sender, receiver | 600 s, fixed | [reference (ref)](LACS-scaip-xml.md#reference-ref), [Message exchange](LACS-scaip-xml.md#message-exchange) |
| Wait for a response (`mrs`) | LUC | 5 s, fixed; up to 3 repetitions with the same `ref` | [Message exchange](LACS-scaip-xml.md#message-exchange) |
| Wait for the response to a command | ARC | 5 s, fixed; up to 3 repetitions with the same `ref` (optional) | [Message exchange](LACS-scaip-xml.md#message-exchange) |
| Response to a camera snapshot command (`aid=0200`) | LUC | at the latest 10 s, fixed | [status-number (snu)](LACS-lacs-xml.md#status-number-snu) |
| Repetition of a voice request after HOLD (`snu=4`) | LUC | every 5 s, fixed | [Fallback Alarm Strategy](LACS-usecases.md#fallback-alarm-strategy-operator-queueing--failover) |
| Effect message after confirming a command | LUC | at most 60 s, fixed | [status-number (snu)](LACS-lacs-xml.md#status-number-snu) |
| `alarm_accept_timeout` | LUC | 180 s, configurable, at most 240 s | [LUC Configuration](#luc-configuration) |
| Call setup after acceptance of a voice alarm | LUC | 30 s, fixed | [Voice call after acceptance fails](LACS-usecases.md#voice-call-after-acceptance-fails) |
| Result report of the voice channel test (`stc` 0207/0203) | LUC | at the latest 30 s after the end of the test call, fixed | [SIP Voice Channel Test Sequence](LACS-usecases.md#sip-voice-channel-test-sequence) |
| Voice channel test call | ARC, LUC | the ARC releases at the latest 30 s after answering; the LUC ends the call at the latest 30 s after it was established | [SIP Voice Channel Test Sequence](LACS-usecases.md#sip-voice-channel-test-sequence) |
| Registration renewal and CRLF keep-alive | LUC | before the registration expires; CRLF keep-alive at the `Flow-Timer` of the registrar, without it at an interval of at most 120 s | [Registration and keep-alive](#registration-and-keep-alive) |
| Re-registration after loss of the connection | LUC | random delay of up to 30 s; at once when an alarm is raised or running | [Registration and keep-alive](#registration-and-keep-alive) |
| Registration expiry | LUC | at most 1800 s, recommended | [Registration and keep-alive](#registration-and-keep-alive) |
| Retry after a SIP error response on the next server address | LUC | at least 10 s, then increasing backoff, each time varied randomly by up to ±10 % (recommended) | [Server addresses and error handling](#server-addresses-and-error-handling) |
| Repetition of an unanswered single event | LUC | increasing backoff (guide value: from 10 s, doubling, up to 1 h), varied randomly by up to ±10 % (recommended) | [Function monitoring and failure notification](LACS-usecases.md#function-monitoring-and-failure-notification) |
| Provisional response (1xx) to an INVITE | ARC | if answering takes longer than 200 ms | [Server addresses and error handling](#server-addresses-and-error-handling) |
| `automatic_test_interval` | LUC | 72 h, configurable, at most 72 h; prEN 81-28:2025 §4.2.8: at least every 3 days | [LUC Configuration](#luc-configuration) |
| Local indication of a failed test transmission | LUC | at the latest 1 h after the first failed attempt (prEN 81-28:2025 §4.2.8) | [Test Transmission](LACS-usecases.md#test-transmission) |
| Time of the test transmission | LUC | random between 90 % and 100 % of `automatic_test_interval` after the previous one (recommended) | [Test Transmission](LACS-usecases.md#test-transmission) |
| Interval of periodic tests | LUC | at least once per `automatic_test_interval`; guide value 24 h or longer | [Function monitoring and failure notification](LACS-usecases.md#function-monitoring-and-failure-notification) |
| Follow-up time of call protection after the end of a session | LUC | implementation-defined | [Call Protection](#call-protection) |
| Wait before a further voice channel test run | LUC | random between 30 s and 120 s, at most 2 further runs (recommended) | [SIP Voice Channel Test Sequence](LACS-usecases.md#sip-voice-channel-test-sequence) |
| Missing test transmission reported to the operator | ARC | `automatic_test_interval` plus 1 h (recommended) | [Test Transmission](LACS-usecases.md#test-transmission) |
| Bypass time of the alarm filter | LUC | adjustable, at most 30 s (prEN 81-28:2025 §4.2.3) | [info-code (ico)](LACS-scaip-xml.md#info-code-ico) |
| Remaining capacity at battery low (`stc` 0016) | LUC | about 5 min of voice communication, guide value | [Long Power Outage](LACS-usecases.md#long-power-outage) |
| Maximum duration of a service voice call | LUC | implementation-defined | [Service voice call](LACS-usecases.md#service-voice-call) |
| Accuracy of the clock of the LUC | LUC | ±5 s, recommended | [event-time (etm)](LACS-scaip-xml.md#event-time-etm) |
| Difference between `etm` and time of receipt shown to the operator | ARC | more than 60 s, recommended | [event-time (etm)](LACS-scaip-xml.md#event-time-etm) |

### Call Protection

The LUC MUST support call protection to prevent unauthorised incoming calls; it is controlled by `call_protection` (default `false`). If call protection is enabled, the LUC accepts an incoming call only

- while an alarm session is active, including a follow-up time after its end (implementation-defined), and
- over its encrypted and authenticated connection to the ARC (TLS connection of its registration, server certificate verified).

All other incoming calls are rejected. The decision does not depend on the `asid`, because intermediate SIP systems can remove it. Calls that the ARC requests by command (callback `aid=0211`, service voice call `aid=0223`) are placed by the LUC and are not affected by call protection.

If the LUC accepts an incoming call that it cannot assign to a running alarm session, it SHOULD play attention tones as for a callback outside an alarm session (see [Voice callback](LACS-usecases.md#voice-callback), rule 6).

This mechanism is analogous to the call protection already implemented in state-of-the-art lift alarm devices for analogue and ISDN connections.

> [!NOTE]
> Call protection is a device-level configuration option. It does not affect the SIP MESSAGE channel, which MUST always remain accessible.

### Authentication and Credentials

LACS uses a public key infrastructure (PKI) for authentication. Passwords can leak and cannot be revoked centrally; certificates avoid both.

1. The LUC and the ARC MUST support TLS (≥ 1.2) with mutual authentication. The LUC authenticates with its operational client certificate (X.509). The subject CN of the certificate is the `controller-id` (`cid`).
2. The ARC MUST verify the client certificate against the CA it trusts for operational certificates and MUST reject a request whose certificate CN does not match the `cid` in the `From` header and in the message.
3. The LUC MUST verify the server certificate against `ca_certificate` and against the name `sip_domain` (RFC 5922), also if `sip_outbound_proxy` is an IP address or another DNS name; the certificate MUST contain `sip_domain` as `dNSName` in its subjectAltName. The LUC resolves a DNS name in `sip_outbound_proxy` with A and AAAA records only; DNS NAPTR and SRV records (RFC 3263) are not used.
4. SIP Digest authentication (RFC 7616) is OPTIONAL. It MAY be used in addition if both LUC and ARC support it; the authentication of the LUC MUST NOT depend on it. This deliberately deviates from TS 50134-9 §4.2.2, which requires HTTP Digest authentication of the LUC.
5. Credentials and keys MUST be unique per device. Default or shared credentials (passwords, keys or certificates used by more than one device) MUST NOT be used. The private key MUST NOT leave the device. Entering secrets manually on site is NOT RECOMMENDED.
6. Operational certificates do not expire, so that nothing on the alarm path runs out: their `notAfter` is `99991231235959Z`, the value that RFC 5280 §4.1.2.5 specifies for a certificate without a well-defined expiration date. The ARC withdraws a device actively by revoking its authorisation. The same applies to the CA certificate in `ca_certificate`. The ARC SHOULD renew its server certificate in good time. A LUC whose clock is not set MUST NOT reject a certificate because of its validity period.
7. How the operational certificate is provisioned to the device is implementation-defined.
8. If the ARC terminates TLS in a front system (e.g. a SIP proxy or session border controller) and passes requests on to internal systems (e.g. a B2BUA), the internal systems MUST accept SIP requests only from the front system, e.g. by checking the source address or by mutual TLS on the internal link. They MUST NOT rely on a header that identifies the LUC alone; otherwise a request sent directly to an internal system bypasses TLS and the check of rule 2.

> [!NOTE]
>
> State of the art is a device identity installed by the manufacturer (e.g. IEEE 802.1AR), with which the device obtains its operational certificate from the ARC automatically by a standard enrolment protocol (e.g. EST, RFC 7030). This avoids manual handling of secrets during installation.

## SIP Profile

### SIP methods

The LUC and the ARC MUST support the SIP methods INVITE (including re-INVITE), REGISTER, ACK, BYE, CANCEL, OPTIONS, UPDATE (RFC 3311) and MESSAGE, and the elementary flows of RFC 3665.

SIP INFO MUST NOT be used, neither for call control nor as keep-alive. Clause 8 of TS 50134-9 (DTMF codes) does not apply; the ARC controls the voice call with `lacs+xml` commands instead (see [Audio control during a voice call](LACS-usecases.md#audio-control-during-a-voice-call)). This deliberately deviates from TS 50134-9 §8.

### Registration and keep-alive

The LUC MUST register with the SIP server of the ARC using REGISTER. Registration is needed so that the ARC can reach the LUC with commands and calls.

1. The LUC MUST register over a persistent TLS connection and keep this connection open. Incoming requests of the ARC (calls and `lacs+xml` commands) arrive over this connection.
2. The LUC MUST keep the connection alive with CRLF keep-alive messages (RFC 5626), at the interval of the `Flow-Timer` of the registrar or, without it, at an interval of at most 120 s; shorter intervals are permitted if the network requires them (e.g. NAT or firewall timeouts). If the connection is lost, the LUC MUST establish a new one and register again after a random delay of up to 30 s; when an alarm is raised or running, at once.
3. The LUC MUST renew its registration before it expires and registers again after it has switched to another server address.
4. The LUC SHOULD use SIP Outbound according to RFC 5626 (`+sip.instance`, `reg-id`, evaluation of the keep-alive response). If it sends `+sip.instance`, the value MUST be unique per device.
5. The ARC MUST accept registrations with and without SIP Outbound, MUST send its requests to the LUC over the connection of its current registration and MUST answer a CRLF keep-alive (double CRLF) with a single CRLF (RFC 5626).

The registration also serves to supervise the LACS alarm path, without a heartbeat message:

- The LUC SHOULD request a registration expiry of at most 1800 s.
- The ARC SHOULD treat the expiry of the registration of a LUC as a failure of its LACS alarm path and inform its operators according to its own procedures. A lost TLS connection alone is not a failure, as long as the LUC registers again before the registration expires.

The ARC thus detects a failure of the alarm path within about 30 min, and not only by a missing test transmission.

### Addressing

The SIP URI of the LUC is `sip:{controller_id}@{sip_domain}`. It is used in the `From` header of all requests sent by the LUC and as address-of-record for the registration. The ARC addresses requests to the LUC (calls and `lacs+xml` commands) to this `sip:` URI, not to a `sips:` URI; they reach the LUC over the TLS connection of its registration.

If a call of the LUC concerns a specific local unit, the `From` header carries the parameter `did={device-id}`, e.g. `From: <sip:{controller_id}@{sip_domain}>;did={device-id}`. In a call to the LUC the ARC names the local unit with the same parameter in the `To` header, e.g. `To: <sip:{controller_id}@{sip_domain}>;did={device-id}`. As `did` consists of token characters only (see [device-id (did)](LACS-scaip-xml.md#device-id-did)), it is carried without quotes.

The LUC calls `tnu` exactly as received from the ARC. If `tnu` is a `sips:` URI, the rules of the secure dialog apply (RFC 3261 §8.1.1.8, §12.1.2): the LUC uses a `sips:` Contact in the INVITE and in all requests within the dialog; the ARC uses `sips:` in its Contact and Record-Route towards the LUC.

### Alert session ID in calls

The LUC and the ARC MUST transmit the `asid` in the SIP header `LACS-ASID` in every INVITE, re-INVITE, UPDATE and BYE of a call that belongs to an alarm session, a voice channel test, a callback or a service voice call, in both directions:

```text
LACS-ASID: {alert-session-id}
```

Session border controllers and other intermediate SIP systems in the path MUST pass this header on. Receivers MUST evaluate the header.

An ARC MAY in addition return its own `tnu` for each alarm session and recognise the incoming call by its Request-URI. The LUC never interprets the content of `tnu` and sends `LACS-ASID` in any case.

### Transport

SIP is carried over TCP with TLS (≥ 1.2); UDP is not used for SIP. The LUC and the ARC SHOULD use TLS 1.3. With TLS 1.2 they SHOULD use only cipher suites with ephemeral key exchange (ECDHE) and authenticated encryption (AEAD, e.g. AES-GCM).

### Server addresses and error handling

`sip_outbound_proxy` can be an IP address or a DNS name. If the DNS name resolves to several addresses, the LUC uses them one after the other.

The following table applies to REGISTER and MESSAGE. For the call to `tnu` see [Voice call after acceptance fails](LACS-usecases.md#voice-call-after-acceptance-fails), for the voice channel test call see [SIP Voice Channel Test Sequence](LACS-usecases.md#sip-voice-channel-test-sequence).

| SIP response | Next action of the LUC |
| ------------ | ---------------------- |
| 3xx          | Follow the redirection |
| 401, 407     | Answer the challenge with SIP Digest if configured; otherwise as for other 4xx |
| 408, 480     | Retry once on the same server address, then as for 5xx |
| other 4xx (e.g. 403, 404) | No retry on the same server address; retry on the next server address after at least 10 s, then keep retrying with increasing backoff time |
| 5xx, 6xx     | Retry on the next server address after at least 10 s, then keep retrying with increasing backoff time |

If a response carries `Retry-After` (RFC 3261 §20.33), the LUC MUST NOT send REGISTER requests, test transmissions or other single events to this server address before that time; it MAY use the next server address meanwhile. Requests of a lift voice alarm do not wait for `Retry-After`: the LUC continues at the next server address after at most 10 s.

The LUC SHOULD vary every backoff time randomly by up to ±10 %, so that many LUCs do not send at the same time, e.g. after a regional power failure.

The ARC MUST send at least one provisional response (1xx) if answering a call takes longer than 200 ms.

### Media profile

The LUC and the ARC MUST support the following media profile, also for early media, with the codec rules below. In its SDP offer the LUC uses this media description (port as allocated), omitting G.729 if it does not support it:

```text
m=audio 40000 RTP/SAVP 8 18 101
a=rtpmap:8 PCMA/8000
a=rtpmap:18 G729/8000
a=fmtp:18 annexb=no
a=rtpmap:101 telephone-event/8000
a=fmtp:101 0-16
a=crypto:1 AES_CM_128_HMAC_SHA1_80 inline:{base64-key}
```

- Codecs: the LUC MUST support G.711 A-law (PCMA) and SHOULD support G.729 without Annex B. The ARC MUST support both, MUST accept an offer with PCMA only and MUST include PCMA in every offer it sends. TS 50134-9 §4.3 requires only one of them; G.729 needs less bandwidth (see [Network Requirements](#network-requirements-informative)).
- `telephone-event` according to RFC 4733.
- SRTP with the crypto suite `AES_CM_128_HMAC_SHA1_80`, keys exchanged in the SDP (SDES, RFC 4568). In addition, the LUC MAY offer `AEAD_AES_128_GCM` (RFC 7714); `AES_CM_128_HMAC_SHA1_80` remains mandatory.
- One audio stream. Further media streams (e.g. text or video) are offered only if the LUC has announced them by `cap`.

Only for a pure fallback call, i.e. a direct call without registration, the LUC MAY fall back to unencrypted RTP (`RTP/AVP`, without `a=crypto`) if no secure connection can be established. Availability then takes precedence over confidentiality.

### NAT traversal

NAT and firewall traversal (ICE, STUN, TURN) is specified in TS 50134-9 §4.1 and §4.3. The ARC MUST use symmetric RTP and RTCP (RFC 4961): it sends media to the source address and port from which it receives the media of the LUC, not only to the address in the SDP. LUCs in mobile networks are often behind carrier-grade NAT.

## Security Considerations

### Requirements of this specification

The following requirements of LACS concern the security of the communication between LUC and ARC:

- TLS (≥ 1.2, TLS 1.3 recommended) over TCP for all SIP signalling (see [Transport](#transport)).
- Mutual authentication with X.509 certificates; credentials unique per device; no default or shared credentials; internal systems behind a TLS-terminating front system of the ARC accept requests from the front system only (see [Authentication and Credentials](#authentication-and-credentials)).
- SRTP (`AES_CM_128_HMAC_SHA1_80`) for voice; unencrypted RTP only for a pure fallback call.
- No SIP INFO: all control of the LUC by the ARC uses `lacs+xml` with `ref`, `asid` and a confirmed response, so that every action is traceable.
- Data minimisation: apart from voice calls, camera images and the text communication of the TBUI, LACS messages carry identifiers, codes and technical status only. They MUST NOT contain personal data of the persons using the lift.

### Relation to IEC 62443-4-2 (informative)

ISO 8102-20 assigns security level 1 (SL1) of IEC 62443-4-2 to the alarm functions of lifts. LACS contributes to the foundational requirements as follows:

| Foundational requirement | Contribution of LACS |
| ------------------------ | -------------------- |
| FR 1 – Identification and authentication control | X.509 client certificates with CN = `cid`; server certificate validation |
| FR 2 – Use control | The ARC authorises each controller; withdrawal by revoking its authorisation |
| FR 3 – System integrity | Periodic tests with explicit results (voice channel, alarm button, microphone/speaker, battery) |
| FR 4 – Data confidentiality | TLS and SRTP; availability of the alarm function takes precedence (fallback call) |
| FR 5 – Restricted data flow | One dedicated connection to the ARC for the alarm function |
| FR 6 – Timely response to events | Every request is answered and identified by `ref` and `asid`; failures are reported |
| FR 7 – Resource availability | Monitoring of the emergency power supply; behaviour on power failure specified |

### Device requirements outside this specification (informative)

The following topics concern the device rather than the protocol and are not specified by LACS. The requirements are given in ISO 8102-20 and the Cyber Resilience Act: secure-by-default configuration (e.g. remote configuration disabled by default), factory reset, software updates, software bill of materials (SBOM), support period, logging of security-relevant events (e.g. change of the ARC address), and documentation for system integration.

## Network Requirements (informative)

For adequate voice and data communication, the IP connectivity of the LUC is expected to provide:

| Parameter        | Value |
| ---------------- | ----- |
| IP versions      | IPv4 and IPv6 |
| Bandwidth (one bidirectional media flow, 20 ms packets, SRTP, IPv4, without layer 2) | G.711 A-law: about 168 kbit/s; G.729: about 56 kbit/s; more with IPv6 |
| Jitter           | expected 100 ms, maximum 200 ms |
| Round-trip delay | maximum 400 ms |
| Packet loss      | below 1 % |
| Availability     | 97 % per year (availability of the transmission network for category SP3 according to EN 50136-1, Table 7) |
| Power backup     | supervised |

Operators are advised to ensure the quality of service on the network segments under the control of the building owner and the service companies and to monitor it on the remaining segments. Connecting the LUC directly to a telecom provider (e.g. via a mobile gateway) avoids routing through the building network and makes adequate power backup easier to ensure.

## Conformance

A conforming **LUC** MUST support:

- the SIP profile, registration and keep-alive and authentication as specified in this document, including call protection,
- the message exchange and the protocol version (`pve`) as specified in [LACS-scaip-xml.md](LACS-scaip-xml.md#message-exchange),
- the `scaip+xml` messages as specified in [LACS-scaip-xml.md](LACS-scaip-xml.md),
- `lacs+xml` as specified in [LACS-lacs-xml.md](LACS-lacs-xml.md) with the commands `aid` 0210 to 0214 (for `0214` presenting the information is optional; the LUC confirms receipt),
- the lift voice alarm including HOLD, reset, the fallback alarm strategy, acknowledgement according to EN 81-28 and end of alarm, the acceptance of a call of the ARC within an alarm session, and the callback requested by `aid=0211` (`stc` 0010 or 0211),
- the announcement of its TBUI and camera functions by `cap`, if it has any,
- the voice channel test including the echo loop,
- test transmission (`stc` 0102), the battery failure report (`stc` 0014), which EN 81-28 requires and which the LUC sends whenever it detects a failure of its emergency power supply, also without a periodic battery test, and the result of the voice channel test (`stc` 0207 or 0203).

OPTIONAL for the LUC, depending on what it can detect or provide:

- further single events (`stc` 0015, 0016, 0023, 0024, 0078, 0085, 0086, 0103, 0112, 0113, 0206),
- further periodic tests with their results (`stc` 0204/0208, 0205/0209, 0014/0210),
- the info codes `ico` 001 and 002,
- the event time (`etm`) and the nameplate elements (`np…`), both RECOMMENDED,
- the electronic lift attendant (`stc` 0990 to 0999),
- the audio control commands (`aid` 0215 to 0219, 0221, 0222, 0224) and the service voice call (`aid` 0223, `stc` 0212),
- text-based alarm communication (TBUI) and camera functions.

A LUC answers a command it does not support with `snu=11`.

A conforming **ARC** MUST support all messages and procedures of this specification, including `lacs+xml`, except the optional extension _Electronic lift attendant_, the service voice call, TBUI and camera functions. This includes sending the audio control commands, although they are optional for the LUC. Holding a voice alarm until an operator has answered (`snu=4`) is OPTIONAL for the ARC. The ARC MUST evaluate the response to every command. Sending "Help is on the way" (`aid=0214`) is RECOMMENDED; towards a LUC with a TBUI it is mandatory (see [Help is on the way](LACS-usecases.md#help-is-on-the-way)).
