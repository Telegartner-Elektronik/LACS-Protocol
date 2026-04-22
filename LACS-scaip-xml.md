# LACS — scaip+xml Message Specification

**Part of:** [Lift Alarm Communication Standard (LACS)](LACS.md) v0.08-draft

## Scope

This document specifies the message properties for `Content-Type: application/scaip+xml` as used within the LACS protocol.

The `scaip+xml` channel covers the standard lift alarm communication as defined by TS 50134-9. It handles all messages initiated by the LUC (Local Unit / Controller) towards the ARC (Alarm Receiving Centre), including voice alarms, test transmissions, status events, and failure notifications.

LACS extensions to `scaip+xml` are intended to be proposed for adoption into the TS 50134-9 standard through ELA and Cenelec.

For optional bidirectional extensions (TBUI, camera, ARC-initiated commands), see [LACS-lacs-xml.md](LACS-lacs-xml.md).
For use cases and sequence diagrams, see [LACS-usecases.md](LACS-usecases.md).

## Message Request (mrq)

### reference (ref)

as specified in TS 50134-9

### media-request (mreq)

| mreq | Description       | Defined by |
| ---- | ----------------- | ---------- |
| 1    | duplex voice call | TS 50134-9 |

For voice alarm (stc=0010) only duplex voice call (mreq=1) is supported. For all other stc the mreq is omitted.

### message-type (mty)

| mty | Description |
| --- | ----------- |
| ME  | Message     |

`PI`, `RE` and `IN` are not used.

### controller-id (cid)

as specified in TS 50134-9

### device-type (dty)

| dty  | Description                                                                  | Defined by |
| ---- | ---------------------------------------------------------------------------- | ---------- |
| 0002 | Unified lift alarm unit and controller (TOC unit with peripherals)           | TS 50134-9 |
| 0032 | Communication link monitor (internal bus between Controller and Local Units) | TS 50134-9 |
| 0041 | Power supply (if it is a separated component)                                | TS 50134-9 |
| 0045 | Lift alarm controller (machine room unit controlling other devices)          | TS 50134-9 |
| 0100 | Top of car unit (if separated from the controller)                           | ELA        |
| 0101 | In car unit / COP unit                                                       | ELA        |
| 0102 | Under-car unit                                                               | ELA        |
| 0110 | Pit unit                                                                     | ELA        |

### device-id (did)

as specified in TS 50134-9

### device-component (dco)

| dco | Description                                        | Defined by |
| --- | -------------------------------------------------- | ---------- |
| 002 | Button 1 (Alarm Button 1)                          | TS 50134-9 |
| 003 | Button 2 (Alarm Button 2)                          | TS 50134-9 |
| 004 | Button 3 (Alarm Button 3)                          | TS 50134-9 |
| 090 | Microphone + speaker                               | LACS       |
| 091 | Camera                                             | LACS       |
| 092 | User interface for text-based communication (TBUI) | LACS       |

### caller-id (crd)

as specified in TS 50134-9

### status-code (stc)

| stc  | Description                                                         | Defined by |
| ---- | ------------------------------------------------------------------- | ---------- |
| 0009 | Alarm triggered by the system                                       | TS 50134-9 |
| 0010 | Alarm, user-triggered - always in combination with voice connection | TS 50134-9 |
| 0014 | Battery failure                                                     | TS 50134-9 |
| 0015 | Battery high/full/charged                                           | TS 50134-9 |
| 0016 | Battery low                                                         | TS 50134-9 |
| 0023 | Communications link, IP, failure                                    | TS 50134-9 |
| 0112 | Communications link, IP, restored                                   | TS 50134-9 |
| 0024 | Comms. link, mobile, failure                                        | TS 50134-9 |
| 0113 | Comms. link mobile, restored                                        | TS 50134-9 |
| 0085 | Power failure (mains)                                               | TS 50134-9 |
| 0086 | Power restored (mains)                                              | TS 50134-9 |
| 0102 | Test transmission / periodic test call                              | TS 50134-9 |
| 0103 | Test transmission, secondary/redundant channel                      | TS 50134-9 |
| 0201 | End of alarm / alert (EOA)                                          | ELA        |
| 0202 | Voice channel (RTP) test - between Controller and ARC               | LACS       |
| 0203 | Voice channel (RTP) failure - between Controller and ARC            | LACS       |
| 0204 | Alarm button failure                                                | LACS       |
| 0205 | Microphone speaker failure                                          | LACS       |
| 0206 | Internal Bus failure                                                | LACS       |

### location-code (lco)

| lco | Description            | Defined by |
| --- | ---------------------- | ---------- |
| 053 | Elevator, lift car     | TS 50134-9 |
| 054 | Elevator, machine room | ELA        |
| 055 | Elevator, top of car   | ELA        |
| 056 | Elevator, pit          | ELA        |

### alert-session-id (asid)

This is an additional property defined in LACS-Protocol, as this is currently missing in TS 50134-9.

Value of the alert-session-id is a 36 char string with an UUID according to RFC 4122.

The alert-session-id will stay stable from start till end of an alert.

|                                                                                     | Start of alert<br />(first usage of unique asid) | End of alert<br />(last usage of same asid) |
| ----------------------------------------------------------------------------------- | ------------------------------------------------ | ------------------------------------------- |
| Lift alarm                                                                          | first mrq with lift voice alarm (stc=0010)       | successful mrs on end of alarm (stc=0201)   |
| Voice channel test                                                                  | mrq with voice channel test (stc=0202)           | SIP BYE                                     |
| Single event transmission <br />independent of an alarm <br />or voice channel test | mrq                                              | successful mrs                              |

Only during a **lift alarm** and a **voice channel test** does the `asid` remain stable across multiple messages. For all other single event transmissions (e.g., battery failure, test transmission, power failure), the `asid` is limited to a single request-response exchange. Each new event transmission requires a new unique `asid`, even if the same condition is being reported again. Implementations must not assume that the `asid` survives a device restart.

**Who creates the asid?**

In the `scaip+xml` context, the `asid` is always created by the LUC, as only the LUC initiates requests.

In the `lacs+xml` context, the ARC may also create an `asid` when it initiates commands independently of an active alarm session. For example, if a periodic test transmission is overdue and the ARC triggers a voice callback (`aid=0211`), the ARC must generate a new `asid` and include it in the outgoing command.

The alert-session-id must be transmitted in every MESSAGE (property `asid` in mrq and mrs). In addition, the `asid` property must also be delivered with each INVITE as a URI parameter in the `Contact` header during voice calls. This means that it is also automatically transferred as a request parameter during BYE.

The `alert-session-id` offers the following advantages:

- The `asid` provides us with a comprehensive, unique identifier for each process. Without asid, we would have to use at least a timestamp (which is not always consistent), controller-id and device-id.
- If the ARC responds to an mrq with status HOLD (snu=4), the LUC must repeat the transmission after a waiting period and also assign a new `ref`. Thanks to the consistent `asid`, the ARC can now determine that this is a repeat and not an independent event.
- The `asid` can be used to uniquely assign all related events to a lift alarm session. This includes, for example, actions in the context of a visual alarm, the retrieval of a camera image from the cabin or the end of alarm.
- The `asid` makes it significantly easier to filter related events when analysing log or trace data.

### capabilities (cap)

comma separated list of 4 digit values

| cap value | Description                                          | Capabilities LUC => ARC                  | Capabilities <br />ARC => LUC |
| --------- | ---------------------------------------------------- | ---------------------------------------- | ----------------------------- |
| 0011      | TBUI supports predefined texts                       | predefined questions + answers           | predefined info (0131)        |
| 0012      | TBUI supports displaying free text received from ARC |                                          | free text (0100, 0101)        |
| 0013      | TBUI supports dynamic answers                        |                                          | dynamic answer options        |
| 0014      | TBUI supports input and sending of free text         | free text (via keyboard or other method) |                               |
| 0021      | CAMERA with still image (single snapshot)            | JPEG snapshot                            |                               |
| 0022      | CAMERA with video stream LUC -> ARC                  | video stream                             |                               |
| 0023      | CAMERA with bidirectional video stream with audio    | audio + video stream                     | audio + video stream          |
| 0900-0999 | Reserved for custom non standard implementations     |                                          |                               |

### custom extensions (x-...)

For flexible use, properties with the prefix "x-" can be defined without notice.

## Message Response (mrs)

### reference (ref)

as specified in TS 50134-9

### status-number (snu)

as specified in TS 50134-9

Important note: on snu=4 the LUC has to retransmit the message with a new ref - but the alarm-session-id (asid) must stay the same

### media-reply (mre)

| mre | Description       | Defined by |
| --- | ----------------- | ---------- |
| 1   | duplex voice call | TS 50134-9 |

For voice alarm (stc=0010) only duplex voice call (mreq=1) is supported. For all other stc the mre is omitted.

### callhandling-reply (cre)

| cre | Description                                                                                  | Defined by |
| --- | -------------------------------------------------------------------------------------------- | ---------- |
| 62  | set-up a voice call to transferred URI/Telephone number defined in transferred-number, <tnu> | TS 50134-9 |

The LUC must support callbacks from the ARC. But the first voice call immediately after the voice alarm (`stc`=0010) is always initiated by the LUC.

Only 62 is allowed here - so the ARC has full control regarding the call routing - if the call to the `tnu` number fails, the LUC should fall back to a predefined number

### transferred-number (tnu)

as specified in TS 50134-9

### alert-session-id (asid)

see above... asid must also be included in the response

### custom extensions (x-...)

For flexible use, properties with the prefix "x-" can be defined without notice.

## Request Overview

| mty | dty                                                              | dco               | stc  | Optional |                                      |
| --- | ---------------------------------------------------------------- | ----------------- | ---- | -------- | ------------------------------------ |
| ME  | 0002 / 0100 / 0101 / <br />0102 / 0103 / 0110                    |                   | 0010 |          | Lift voice alarm                     |
| ME  | 0002 / 0100 / 0101 / <br />0102 / 0103 / 0110                    |                   | 0201 |          | End of Alarm                         |
| ME  | 0002 / 0041 /0045 / <br />0100 / 0101 / 0102 / <br />0103 / 0110 |                   | 0014 |          | Battery failure                      |
| ME  | 0002 / 0041 /0045 / <br />0100 / 0101 / 0102 / <br />0103 / 0110 |                   | 0015 |          | Battery high/full/recharged          |
| ME  | 0002 / 0041 /0045 / <br />0100 / 0101 / 0102 / <br />0103 / 0110 |                   | 0016 |          | Battery low                          |
| ME  | 0002 / 0041 /0045 / <br />0100 / 0101 / 0102 / <br />0103 / 0110 |                   | 0085 |          | Power failure (mains)                |
| ME  | 0002 / 0041 /0045 / <br />0100 / 0101 / 0102 / <br />0103 / 0110 |                   | 0086 |          | Power restored (mains)               |
| ME  | 0002 / 0100 / 0101 / <br />0102 / 0103 / 0110                    | 002 / 003 / 004   | 0204 | X        | Alarm button failure                 |
| ME  | 0002 / 0100 / 0101 / <br />0102 / 0103 / 0110                    | 090 (Mic+Speaker) | 0205 | X        | Microphone speaker failure           |
| ME  | 0032 (Comm link monitor)                                         | -                 | 0206 | X        | Internal Bus failure                 |
| ME  | 0045 (Controller)                                                |                   | 0203 | X        | Communication voice failure          |
| ME  | 0002 / 0045 / 0100 / <br />0101 / 0102 / 0103 / <br />0110       |                   | 0102 |          | Test transmission, primary channel   |
| ME  | 0002 / 0045 / 0100 / <br />0101 / 0102 / 0103 / <br />0110       |                   | 0103 |          | Test transmission, redundant channel |
