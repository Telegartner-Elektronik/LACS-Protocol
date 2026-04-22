# LACS — lacs+xml Message Specification

**Part of:** [Lift Alarm Communication Standard (LACS)](LACS.md) v0.08-draft

## Scope

This document specifies the message properties for `Content-Type: application/lacs+xml` as used within the LACS protocol.

The `lacs+xml` channel provides optional extensions for advanced bidirectional communication between the ARC (Alarm Receiving Centre) and the LUC (Local Unit / Controller). This includes:

- **ARC-initiated commands** (CO) — e.g., trigger test transmission, trigger voice callback, request camera snapshot, trigger end of alarm
- **Text-based alarm communication** (TD) — TBUI dialogue actions and messages between trapped passengers and ARC operators

For the standard lift alarm communication channel, see [LACS-scaip-xml.md](LACS-scaip-xml.md).
For use cases and sequence diagrams, see [LACS-usecases.md](LACS-usecases.md).

## Message Request (mrq)

### reference (ref)

same as in TS 50134-9

### message-type (mty)

| mty | Description                   |
| --- | ----------------------------- |
| CO  | Command                       |
| TD  | TBUI dialog action or message |

### action-id (aid)

action-id must always be interpreted in combination with `mty`

| mty | aid            | Direction   | Description                                      |
| --- | -------------- | ----------- | ------------------------------------------------ |
| TD  | 0100 (default) | LUC <=> ARC | Free text                                        |
| TD  | 0101           | ARC => LUC  | Free text info (no answer expected)              |
| TD  | 0109           | LUC => ARC  | Set Language (ISO 639-1 code in txt)             |
| TD  | 0111           | LUC => ARC  | Auto Question 1 "Do you need help"               |
| TD  | 0112           | LUC => ARC  | Auto Question 2 "Do you need an ambulance?"      |
| TD  | 0121           | LUC => ARC  | Answer "yes"                                     |
| TD  | 0122           | LUC => ARC  | Answer "no"                                      |
| TD  | 0131           | ARC => LUC  | Info 1 (no answer expected) "Help is on the way" |
| CO  | 0200           | ARC => LUC  | Get Camera image (snapshot)                      |
| CO  | 0210           | ARC => LUC  | Trigger test transmission                        |
| CO  | 0211           | ARC => LUC  | Trigger voice callback                           |
| CO  | 0212           | ARC => LUC  | Trigger end of alarm                             |
|     | 0900 - 0999    |             | Reserved for flexible use without notice         |

### controller-id (cid)

same as in TS 50134-9

### device-id (did)

same as in TS 50134-9

### device-component (dco)

| dco | Description                                        | Defined by |
| --- | -------------------------------------------------- | ---------- |
| 002 | Button 1 (Alarm Button 1)                          | TS 50134-9 |
| 003 | Button 2 (Alarm Button 2)                          | TS 50134-9 |
| 004 | Button 3 (Alarm Button 3)                          | TS 50134-9 |
| 090 | Microphone + speaker                               | LACS       |
| 091 | Camera                                             | LACS       |
| 092 | User interface for text-based communication (TBUI) | LACS       |

### alert-session-id (asid)

see definition under [scaip+xml mrq](LACS-scaip-xml.md#alert-session-id-asid)

### text (txt)

Must always be interpreted in combination with `mty` and `aid`.

Text is always encoded as UTF-8

### custom extensions (x-...)

For flexible use, properties with the prefix "x-" can be defined without notice.

## Message Response (mrs)

### reference (ref)

same as in TS 50134-9

### status-number (snu)

same as in TS 50134-9

Important note: on `snu`=4 the LUC has to re-transmit the message with a new ref - but the alarm-session-id (asid) must stay the same

### custom extensions (x-...)

For flexible use, properties with the prefix "x-" can be defined without notice.
