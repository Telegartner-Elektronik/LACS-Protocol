# LACS — lacs+xml Message Specification

**Part of:** [Lift Alarm Communication Standard (LACS)](LACS.md), see there for version and status

## Scope

This document specifies the message properties for `Content-Type: application/lacs+xml` as used within the LACS protocol.

The `lacs+xml` channel carries the bidirectional communication between the ARC (Alarm Receiving Centre) and the LUC (Local Unit / Controller) beyond TS 50134-9. The LUC and the ARC MUST support it (see [Conformance](LACS.md#conformance)). It includes:

- **ARC-initiated commands** (CO) — e.g., trigger test transmission, trigger voice callback, request camera snapshot, trigger end of alarm, "help is on the way", audio control during a voice call, service voice call
- **Text-based alarm communication** (TD) — TBUI dialogue actions and messages between trapped passengers and ARC operators (optional, announced by `cap`)

Receivers MUST ignore elements and values they do not know (TS 50134-9 §7.2.2). The XML schema [xml-schema/lacs_lacs-xml.xsd](xml-schema/lacs_lacs-xml.xsd) checks the syntax; valid code values are those in the tables below.

For the standard lift alarm communication channel, see [LACS-scaip-xml.md](LACS-scaip-xml.md).
For procedures, sequence diagrams and examples, see [LACS-usecases.md](LACS-usecases.md).

The rules of [Message exchange](LACS-scaip-xml.md#message-exchange) apply to `lacs+xml` as well.

## Message Request (mrq)

### reference (ref)

see definition under [scaip+xml mrq](LACS-scaip-xml.md#reference-ref)

### protocol-version (pve)

see definition under [scaip+xml mrq](LACS-scaip-xml.md#protocol-version-pve)

### message-type (mty)

| mty | Description                   | Defined by |
| --- | ----------------------------- | ---------- |
| CO  | Command                       | LACS       |
| TD  | TBUI dialog action or message | LACS       |

### action-id (aid)

The receiver MUST interpret `aid` in combination with `mty`.

| mty | aid            | Direction   | Description                                      |
| --- | -------------- | ----------- | ------------------------------------------------ |
| TD  | 0100           | LUC <=> ARC | Free text (experimental); assumed if `aid` is missing in a `TD` message |
| TD  | 0101           | ARC => LUC  | Free text info, no answer expected (experimental) |
| TD  | 0109           | LUC => ARC  | Set Language (ISO 639-1 code in txt)             |
| TD  | 0111           | LUC => ARC  | Auto Question 1 "Do you need help"               |
| TD  | 0112           | LUC => ARC  | Auto Question 2 "Do you need an ambulance?"      |
| TD  | 0121           | LUC => ARC  | Answer "yes"                                     |
| TD  | 0122           | LUC => ARC  | Answer "no"                                      |
| CO  | 0200           | ARC => LUC  | Get Camera image (snapshot)                      |
| CO  | 0210           | ARC => LUC  | Trigger test transmission (LUC sends `stc` 0102) |
| CO  | 0211           | ARC => LUC  | Trigger voice callback (LUC sends `stc` 0010 or 0211 and calls) |
| CO  | 0212           | ARC => LUC  | Trigger end of alarm                             |
| CO  | 0213           | ARC => LUC  | Trigger voice channel test (LUC runs `stc` 0202 with the `asid` of the command; while an alarm is open at the controller `snu=12`, see [General rules](LACS-usecases.md#general-rules)) |
| CO  | 0214           | ARC => LUC  | Help is on the way (see [Help is on the way](LACS-usecases.md#help-is-on-the-way)) |
| CO  | 0215           | ARC => LUC  | Local unit speaker volume down (-) - during an active voice call only |
| CO  | 0216           | ARC => LUC  | Local unit speaker volume up (+) - during an active voice call only |
| CO  | 0217           | ARC => LUC  | Duplex voice mode - during an active voice call only |
| CO  | 0218           | ARC => LUC  | Simplex voice mode, microphone (local unit speaker muted) - during an active voice call only |
| CO  | 0219           | ARC => LUC  | Simplex voice mode, speaker (local unit microphone muted) - during an active voice call only |
| CO  | 0221           | ARC => LUC  | Local unit microphone sensitivity down (-) - during an active voice call only |
| CO  | 0222           | ARC => LUC  | Local unit microphone sensitivity up (+) - during an active voice call only |
| CO  | 0223           | ARC => LUC  | Trigger service voice call (LUC sends `stc` 0212 and calls) |
| CO  | 0224           | ARC => LUC  | Select voice unit (`dco`) - during an active voice call only |
|     | 0900 - 0999    |             | Reserved for custom non-standard use             |

`aid` 0220 is reserved and not used in LACS.

The effect of the audio control commands (`0215` to `0219`, `0221`, `0222`, `0224`) ends with the call, except for settings changed in a service voice call. See [Audio control during a voice call](LACS-usecases.md#audio-control-during-a-voice-call) and [Service voice call](LACS-usecases.md#service-voice-call).

### controller-id (cid)

see definition under [scaip+xml mrq](LACS-scaip-xml.md#controller-id-cid)

### device-id (did)

see definition under [scaip+xml mrq](LACS-scaip-xml.md#device-id-did)

### internal-bus-address (iba)

see definition under [scaip+xml mrq](LACS-scaip-xml.md#internal-bus-address-iba). In addition, for commands:

- A command addressed to a local unit MUST contain `did`. The ARC MAY add the `iba` configured for this `did`; it MUST NOT send a command with `iba` but without `did`.
- If `did` and `iba` do not denote the same local unit, or the `iba` is unknown, the LUC MUST answer `snu=3` and MUST NOT carry out the command.

### device-component (dco)

see definition under [scaip+xml mrq](LACS-scaip-xml.md#device-component-dco)

### alert-session-id (asid)

see definition under [scaip+xml mrq](LACS-scaip-xml.md#alert-session-id-asid)

### text (txt)

The receiver MUST interpret `txt` in combination with `mty` and `aid`.

Text is always encoded as UTF-8, 1 to 256 characters.

### custom extensions (x-...)

see definition under [scaip+xml mrq](LACS-scaip-xml.md#custom-extensions-x-)

## Message Response (mrs)

An `mrs` is always sent in the content type of its `mrq`: the LUC answers a command (`application/lacs+xml`) with an `mrs` in `application/lacs+xml`. The only exception is the response to a camera snapshot (`aid=0200`): it is sent as `multipart/mixed` and contains the `mrs` in `application/lacs+xml` and the JPEG image (see [Single Snapshot](LACS-usecases.md#single-snapshot)).

### reference (ref)

see definition under [scaip+xml mrq](LACS-scaip-xml.md#reference-ref)

### protocol-version (pve)

see definition under [scaip+xml mrq](LACS-scaip-xml.md#protocol-version-pve)

### status-number (snu)

see definition under [scaip+xml mrs](LACS-scaip-xml.md#status-number-snu). For responses to commands (`CO`):

1. If the LUC does not support a command, it MUST answer `snu=11` with `ste` `Error-unsupported-action`.
2. If the LUC supports a command but cannot carry it out in its current state (e.g. an audio control command while no voice call is established), it MUST answer `snu=12` with `ste` `Error-not-permitted-in-state` and MUST NOT carry out the command.
3. If the `asid` of a command that requires a running session (not `0210`, `0211`, `0213` and `0223`, which the ARC can also send outside a session) is unknown to the LUC (e.g. after a restart), the LUC MUST answer `snu=3` with `ste` `Error-unknown-asid` and MUST NOT carry out the command.
4. The ARC MUST evaluate the response to every command. If the response is not `snu=0`, the ARC MUST indicate to the operator that the command has not been carried out.
5. The `mrs` only confirms receipt, except for the camera snapshot (`aid=0200`), whose `mrs` carries the image. The LUC MUST answer a command at once, the camera snapshot as soon as the image is available and at the latest 10 s after receipt. Besides `snu` 0, 3, 11 and 12 it can use the error numbers of TS 50134-9 (e.g. 1, 2, 7, 99); it MUST NOT answer a command with `snu=4`. The ARC MUST NOT repeat a confirmed command. The effect of a command becomes visible as a message of the LUC.
6. For commands with an effect message (`0210`: test transmission `stc` 0102, `0211`: voice request `stc` 0010 or 0211, `0212`: end of alarm `stc` 0201, `0213`: start of voice channel test `stc` 0202, `0223`: voice request `stc` 0212), the LUC MUST send this message at the latest 60 s after the confirmation. If it cannot carry out a confirmed command (e.g. the local unit cannot be reached), it does not send the message. If the message does not arrive within 60 s, the ARC MUST indicate to the operator that the command has not been carried out.

### custom extensions (x-...)

see definition under [scaip+xml mrq](LACS-scaip-xml.md#custom-extensions-x-)
