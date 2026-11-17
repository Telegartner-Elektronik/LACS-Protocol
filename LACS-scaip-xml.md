# LACS — scaip+xml Message Specification

**Part of:** [Lift Alarm Communication Standard (LACS)](LACS.md), see there for version and status

## Scope

This document specifies the message properties for `Content-Type: application/scaip+xml` as used within the LACS protocol.

The `scaip+xml` channel covers the standard lift alarm communication as defined by TS 50134-9. It handles all messages initiated by the LUC (Local Unit / Controller) towards the ARC (Alarm Receiving Centre), including voice alarms, test transmissions, status events, and failure notifications.

The following additions of LACS to `scaip+xml` are intended to be proposed to CENELEC via ELA for adoption into TS 50134-9: the status codes 0202 to 0210, the info codes 001 and 002 and the elements `asid`, `iba`, `pve`, `etm` and the nameplate elements `np…`. The additions published by ELA (e.g. status code 0201, device types 0100 to 0110, location codes 054 to 056) are proposed by ELA. Not proposed are the codes and elements that serve `lacs+xml` (`cap`, device components 030 and 031, status codes 0211 and 0212, status numbers 11 and 12) and the codes of the electronic lift attendant (0990 to 0999), because they are locally limited.

Receivers MUST ignore elements and values they do not know (TS 50134-9 §7.2.2). The XML schema [xml-schema/lacs_scaip-xml.xsd](xml-schema/lacs_scaip-xml.xsd) checks the syntax; valid code values are those in the tables below.

For the commands of the ARC and the extensions TBUI and camera, see [LACS-lacs-xml.md](LACS-lacs-xml.md).
For procedures, sequence diagrams and examples, see [LACS-usecases.md](LACS-usecases.md).

## Message exchange

A message request (`mrq`) is answered by exactly one message response (`mrs`) with the same `ref`. The following rules apply to `scaip+xml` and `lacs+xml`:

1. If no `mrs` arrives within 5 s, the LUC MUST repeat the request with the same `ref` and the same `asid`, up to 3 times. It then continues with the next server address (see [Server addresses and error handling](LACS.md#server-addresses-and-error-handling)), finally, for a lift voice alarm only, with a fallback alarm path, if the LUC has one. A repetition after HOLD (`snu=4`) is different: the request has been answered and is repeated with a new `ref` (see [reference (ref)](#reference-ref)).
2. A receiver that gets a request with a `ref` it has already answered within 600 s (same `cid`, `mty` and `ref`) MUST repeat its earlier response and MUST NOT process the request again. A repetition of a request that the receiver is still processing (e.g. a camera snapshot) is ignored; the receiver sends its response once.
3. If the ARC receives no `mrs` to a command within 5 s, it MAY repeat the command with the same `ref` up to 3 times. If there is still no response, it MUST indicate to the operator that the command has not been carried out.
4. A sender MUST discard every `mrs` whose `ref` does not belong to a request it currently has open. This keeps a late response harmless. After the LUC has sent a reset (`RE`), an `mrs` with the `ref` of the reset belongs to the reset.
5. A sender MUST send every `mrq` and every `mrs` as one SIP MESSAGE. LACS does not split long messages by `additional-message` (`ame`), because SIP is carried over TCP with TLS only (see [Transport](LACS.md#transport)); this deliberately deviates from TS 50134-9 Table 1. A sender MUST NOT send `ame`; a receiver MUST ignore it. The body of a MESSAGE MUST NOT exceed 4096 bytes, except for the response to a camera snapshot (see [Single Snapshot](LACS-usecases.md#single-snapshot)). A receiver MUST answer a request with a larger body with `snu=1` (message too long).
6. A receiver MUST answer a request with an unsupported `mty` with `snu=99` and `ste` `Error-unsupported-message`, as required by TS 50134-9 §7.2.2.

## Message Request (mrq)

### reference (ref)

as specified in TS 50134-9. In addition, a sender MUST NOT reuse a `ref` for a different request within 600 s, also across a restart (e.g. a random value or a counter that survives a restart). A repetition after HOLD (`snu=4`) gets a new `ref` and keeps the `asid`. This deliberately deviates from TS 50134-9 §5.2, which requires the same Message Request to be resent: the `asid` identifies the repetition, and every request remains individually traceable. A reset (`RE`) uses the `ref` of the first request that the LUC has sent in this session to this ARC since its last reset there (TS 50134-9 §5.4); this is not a reuse.

### protocol-version (pve)

Protocol and version of LACS that the sender implements, in the form `LACS/MAJOR.MINOR`: the protocol name `LACS` (upper case), a slash and `MAJOR.MINOR` of the version in the header of [LACS.md](LACS.md), without PATCH and suffix (see [Versioning](LACS.md#versioning)), e.g. `LACS/1.0` for 1.0.0-rc1 and 1.0.2, or `LACS/1.2` for 1.2.0. Each part of the version has 1 to 3 digits without leading zeros. The form follows the protocol versions of SIP and HTTP (`SIP/2.0`, `HTTP/1.1`). Defined by LACS. The same rules apply to `lacs+xml`.

1. Every sender MUST include `pve` in every `mrq` and every `mrs`.
2. A receiver MUST accept a message without `pve` and treats it as `LACS/1.0`.
3. In the `mrs` the responder MUST send the highest version it supports with the same major version as in the `mrq`.
4. If the receiver does not support the major version, it answers `snu=99` with `ste` `Error-unsupported-version` and its highest supported version in `pve`; this deliberately deviates from TS 50134-9 §7.2.3, which names the highest supported major version below the requested one. The same applies if `pve` names a protocol other than `LACS` or does not have the form above. If the LUC also supports the major version reported, it SHOULD repeat the request with this version, a new `ref` and the same `asid`; otherwise it proceeds as for `snu=5`.
5. A side MAY use a function introduced with a minor version only if the other side has reported at least this version. The LUC learns the version of the ARC from the `mrs`, the ARC the version of the LUC from the `mrq`. Each side keeps the version last reported by the other side. A LUC still answers a command it does not know with `snu=11`.
6. LACS does not use `ver` of the request and `cve` of the response of TS 50134-9. A sender SHOULD NOT send `ver` and MUST NOT send `cve`; a receiver ignores both. `pve` takes their place: this deliberately deviates from TS 50134-9 §7.2.1 to §7.2.3. The rules on major and minor versions, on ignoring unknown elements (§7.2.2) and on `snu=99` remain the same in substance. A later edition of TS 50134-9 used by LACS follows from the LACS version.

### media-request (mreq)

| mreq | Description       | Defined by |
| ---- | ----------------- | ---------- |
| 1    | duplex voice call | TS 50134-9 |

The LUC MUST send `mreq=1` with every request that asks for a voice call: `stc` 0010 (lift voice alarm), 0202 (start of voice channel test), 0211 (callback) and 0212 (service voice call). For all other `stc` it MUST omit `mreq`.

### message-type (mty)

| mty | Description | Defined by |
| --- | ----------- | ---------- |
| ME  | Message     | TS 50134-9 |
| RE  | Reset       | TS 50134-9 §5.4 |

`PI` and `IN` are not used. LACS has no heartbeat; this deliberately deviates from TS 50134-9 §5.5. The registration of the LUC, which is needed anyway so that the ARC can reach it, supervises the alarm path (see [Registration and keep-alive](LACS.md#registration-and-keep-alive)); the test transmission and the voice channel test according to EN 81-28 complete it.

In addition to TS 50134-9 §5.4, LACS requires for `RE`:

- the `asid` of the session being reset MUST be included,
- the ARC MUST respond with `snu=0`, also if the `asid` is unknown.

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

1 to 16 characters, consisting of letters (`A`–`Z`, `a`–`z`), digits and the characters `-`, `.`, `_` and `~` only. This restricts TS 50134-9, which allows all printable US-ASCII characters: `did` is a token according to RFC 3261 and is carried as a parameter in SIP headers (see [Addressing](LACS.md#addressing)). The LUC MUST send `did` if the message concerns a specific local unit (e.g. lift voice alarm, end of alarm, test transmission, alarm button or microphone/speaker test); for messages of the controller itself it MAY omit it. A receiver MUST NOT make up a missing `did`.

### internal-bus-address (iba)

Address of the local unit on the internal bus of the controller, 1 to 9999 without leading zeros. A controller has exactly one internal bus, so `iba` is unique per `cid`. Defined by LACS.

- If the controller has more than one local unit, the LUC MUST send `iba` in every message that contains `did`.
- If it has only one local unit, the LUC MAY omit `iba`; it then counts as `iba=1`.
- `did` remains the identifier of the local unit; `iba` never appears without `did`.

### device-component (dco)

| dco | Description                                        | Defined by |
| --- | -------------------------------------------------- | ---------- |
| 002 | Alarm Button / Voice Unit 1                        | TS 50134-9 |
| 003 | Alarm Button / Voice Unit 2                        | TS 50134-9 |
| 004 | Alarm Button / Voice Unit 3                        | TS 50134-9 |
| 005 | Alarm Button / Voice Unit 4                        | TS 50134-9 |
| 006 | Alarm Button / Voice Unit 5                        | TS 50134-9 |
| 030 | Camera                                             | LACS       |
| 031 | User interface for text-based communication (TBUI) | LACS       |

The codes 002 to 006 ("Button 1" to "Button 5" in TS 50134-9 Table B.1) each denote an alarm button together with the voice unit (microphone and speaker) at the same location, e.g. in the car, on the car roof or in the pit. A local unit with one alarm button and one voice unit uses `002`. A message that concerns an alarm button or a voice unit MAY carry its location in `lco`.

The codes 030 and 031 lie in the range that TS 50134-9:2018 Table B.1 allocates for future use (030–089).

### caller-id (crd)

as specified in TS 50134-9

### status-code (stc)

In a message (`mty=ME`) the LUC MUST send `stc`. This deliberately deviates from TS 50134-9 Table 1, which assumes `0010` if `stc` is missing.

The table is grouped by function: alarm and calls, power supply, communication links and internal bus, tests.

| stc  | Description                                                         | Defined by |
| ---- | ------------------------------------------------------------------- | ---------- |
| 0010 | Alarm, user-triggered - always in combination with voice connection | TS 50134-9 |
| 0211 | Voice call requested by the ARC (callback)                          | LACS       |
| 0212 | Service voice call requested by the ARC                             | LACS       |
| 0201 | End of alarm / alert (EOA)                                          | ELA        |
| 0085 | Power failure (mains)                                               | TS 50134-9 |
| 0086 | Power restored (mains)                                              | TS 50134-9 |
| 0014 | Battery failure - also the failed result of a periodic battery test (counterpart of `0210`) | TS 50134-9 |
| 0210 | Battery test passed                                                 | LACS       |
| 0015 | Battery high/full/charged                                           | TS 50134-9 |
| 0016 | Battery low - remaining capacity for about 5 minutes of voice communication (guide value) | TS 50134-9 |
| 0078 | Out of Service - the LUC switches off because its emergency power supply is depleted; alarms can no longer be transmitted | TS 50134-9 |
| 0023 | Communications link, IP, failure                                    | TS 50134-9 |
| 0112 | Communications link, IP, restored                                   | TS 50134-9 |
| 0024 | Comms. link, mobile, failure                                        | TS 50134-9 |
| 0113 | Comms. link mobile, restored                                        | TS 50134-9 |
| 0206 | Internal Bus failure                                                | LACS       |
| 0102 | Test transmission / periodic test call                              | TS 50134-9 |
| 0103 | Test transmission, secondary/redundant channel                      | TS 50134-9 |
| 0202 | Start of voice channel (RTP) test - between Controller and ARC      | LACS       |
| 0207 | Voice channel (RTP) test passed                                     | LACS       |
| 0203 | Voice channel (RTP) failure - between Controller and ARC            | LACS       |
| 0208 | Alarm button test passed                                            | LACS       |
| 0204 | Alarm button failure                                                | LACS       |
| 0209 | Microphone speaker test passed                                      | LACS       |
| 0205 | Microphone speaker failure                                          | LACS       |

The codes 0201 to 0212 lie in the range that TS 50134-9:2018 Table C.1 allocates for future use (0128–0899); for the codes proposed for adoption into TS 50134-9 see [Scope](#scope). For the use of 0202 to 0210 see [Function monitoring and failure notification](LACS-usecases.md#function-monitoring-and-failure-notification), for 0211 and 0212 see [Voice callback](LACS-usecases.md#voice-callback) and [Service voice call](LACS-usecases.md#service-voice-call). The status codes of the electronic lift attendant are specified separately, see [Optional extension: Electronic lift attendant](#optional-extension-electronic-lift-attendant).

### location-code (lco)

| lco | Description            | Defined by |
| --- | ---------------------- | ---------- |
| 053 | Elevator, lift car     | TS 50134-9 |
| 054 | Elevator, machine room | ELA        |
| 055 | Elevator, top of car   | ELA        |
| 056 | Elevator, pit          | ELA        |

### info-code (ico)

Optional supplement to `stc` (TS 50134-9 Annex E). LACS uses it only for the lift voice alarm:

| ico | stc  | Description                                                                                                                                                      | Defined by |
| --- | ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- |
| 001 | 0010 | Alarm filter bypassed: a filter criterion applied (prEN 81-28:2025 §4.2.3) and the alarm button was pressed continuously for the configured bypass time (≤ 30 s) | LACS       |
| 002 | 0010 | Alarm filtering enabled (prEN 81-28:2025 §4.2.3), but no filter criterion applied when the alarm was initiated | LACS |

A LUC with alarm filtering enabled SHOULD send `ico=001` or `ico=002` with every lift voice alarm. Without `ico` the LUC gives no information about alarm filtering, e.g. because it does not implement filtering or filtering is not enabled. An alarm with `ico=001` typically indicates a test by maintenance staff. Whatever the `ico`, the ARC MUST handle the alarm like any other alarm. The optional `info-text` (`ite`, 1 to 128 characters) MAY carry additional context (e.g. `button held 25 s, car running`) and MUST NOT contradict `ico`.

### status-text (stt)

as specified in TS 50134-9: 1 to 32 characters, UTF-8, informative. In a repeated voice request the LUC MAY state in `stt` why the previous call failed, e.g. `Call failed: 486 Busy Here`.

### event-time (etm)

Time at which the reported event occurred, according to RFC 3339 in UTC with `Z`, with seconds and optional milliseconds, e.g. `2026-11-04T09:15:02Z` or `2026-11-04T09:15:02.250Z`. Defined by LACS. Only in an `mrq` with `mty=ME`; not in commands and not in the `mrs`.

1. The LUC SHOULD include `etm` in every `mrq` with `mty=ME` if its clock is synchronised.
2. If its clock is not synchronised (e.g. after a restart without time source), the LUC MUST NOT include `etm`. A missing time is better than a wrong one.
3. If the LUC sends a message more than 5 s after the event occurred (buffering, repetition, change to another ARC, restart), it MUST include `etm`, provided its clock is synchronised.
4. `etm` is the time of the event, not of sending. A repetition (with the same or a new `ref`) carries the same `etm` as the first transmission. Pressing the alarm button again is a new event with its own `etm`.
5. The LUC SHOULD synchronise its clock, e.g. by NTP or from the mobile network, with an accuracy of ±5 s.
6. The ARC MUST use the time of receipt for all procedures and timers. `etm` is information for the operator and the log.
7. The ARC SHOULD store `etm` together with the time of receipt and SHOULD show it to the operator if the two differ by more than 60 s.
8. Without `etm` the time of receipt applies.

LACS does not use `tim` within `lge` of TS 50134-9, because it belongs to the geographical position and allows time zone offsets in whole hours only. A receiver ignores it (TS 50134-9 §7.2.2).

### alert-session-id (asid)

Defined by LACS; TS 50134-9 has no such element.

Value of the alert-session-id is a UUID according to RFC 9562 (versions 1 to 8) in its 36-character string form.

The `asid` stays the same from the start to the end of a session.

|                                                                                     | Start of alert<br />(first usage of unique asid) | End of alert<br />(last usage of same asid) |
| ----------------------------------------------------------------------------------- | ------------------------------------------------ | ------------------------------------------- |
| Lift alarm                                                                          | first mrq with lift voice alarm (stc=0010)       | successful mrs on end of alarm (stc=0201), or on a reset (`RE`) at this ARC |
| Voice channel test                                                                  | mrq with start of voice channel test (stc=0202), or command `aid=0213` of the ARC | successful mrs on the test result (stc=0207 or 0203), or on a reset (`RE`) of the test |
| Test transmission requested by the ARC                                              | command `aid=0210` of the ARC                    | successful mrs on the test transmission (stc=0102) |
| Callback outside an alarm                                                           | command `aid=0211` of the ARC                    | end of the call                             |
| Service voice call                                                                  | command `aid=0223` of the ARC                    | end of the call                             |
| Single event transmission <br />independent of an alarm <br />or voice channel test | mrq                                              | successful mrs                              |

Only in the sessions listed above (lift alarm, voice channel test, and the sessions opened by the commands `aid` 0210, 0211, 0213 and 0223) does the `asid` remain stable across multiple messages. For all other single event transmissions (e.g., battery failure, test transmission, power failure), the `asid` belongs to one event; repetitions of the same report keep it. Each new event requires a new unique `asid`, even if the same condition is being reported again. Implementations MUST NOT assume that the `asid` survives a device restart. The LUC SHOULD keep the `asid` of a running lift voice alarm in non-volatile memory, so that the end of alarm carries the right `asid` after a restart.

**Who creates the asid?**

In the `scaip+xml` context, the `asid` is always created by the LUC, as only the LUC initiates requests.

In the `lacs+xml` context, the ARC MAY send the commands `aid=0210` (trigger test transmission), `aid=0211` (trigger voice callback), `aid=0213` (trigger voice channel test) and `aid=0223` (trigger service voice call) outside a running session. The ARC MUST then create a new `asid` and include it in the command. For example, if a test transmission is overdue, the ARC triggers a test transmission (`aid=0210`) with a new `asid`; the LUC sends the test transmission with this `asid`.

The sender MUST include the `asid` in every MESSAGE (`mrq` and `mrs`). In addition, it is transmitted in the SIP header `LACS-ASID` of every INVITE, re-INVITE, UPDATE and BYE of a call belonging to the session (see [Alert session ID in calls](LACS.md#alert-session-id-in-calls)).

The `alert-session-id` offers the following advantages:

- The `asid` provides a comprehensive, unique identifier for each process. Without asid, we would have to use at least a timestamp (which is not always consistent), controller-id and device-id.
- If the ARC responds to an mrq with status HOLD (snu=4), the LUC repeats the transmission after a waiting period and assigns a new `ref`. Thanks to the consistent `asid`, the ARC can now determine that this is a repeat and not an independent event.
- The `asid` can be used to uniquely assign all related events to a lift alarm session. This includes, for example, actions in the context of a visual alarm, the retrieval of a camera image from the cabin or the end of alarm.
- The `asid` makes it significantly easier to filter related events when analysing log or trace data.

### capabilities (cap)

comma separated list of 4 digit values

| cap value | Description                                          | Capabilities LUC => ARC                  | Capabilities <br />ARC => LUC |
| --------- | ---------------------------------------------------- | ---------------------------------------- | ----------------------------- |
| 0011      | TBUI supports predefined texts                       | predefined questions + answers           | predefined info (0214)        |
| 0012      | TBUI supports displaying free text received from ARC (experimental) |                                          | free text (0100, 0101)        |
| 0013      | TBUI supports dynamic answers (experimental)         |                                          | dynamic answer options        |
| 0014      | TBUI supports input and sending of free text (experimental) | free text (via keyboard or other method) |                               |
| 0015      | TBUI supports real-time text (RFC 4103) (experimental) | real-time text                          | real-time text                |
| 0021      | CAMERA with still image (single snapshot)            | JPEG snapshot                            |                               |
| 0022      | CAMERA with video stream LUC -> ARC (experimental)   | video stream                             |                               |
| 0023      | CAMERA with bidirectional video stream with audio (experimental) | audio + video stream                     | audio + video stream          |
| 0900-0999 | Reserved for custom non standard implementations     |                                          |                               |

If the local unit of the message has at least one of these functions, the LUC MUST send `cap` in every voice request (`stc` 0010, 0211, 0212) and in every test transmission (`stc` 0102), listing all of them. The ARC thus knows the functions also for a callback and a service voice call. Without `cap` the ARC assumes that the local unit has none of these functions.

### nameplate (np…)

Nameplate data of the controller (`cid`) and of the local unit (`did`) of the message: the identification data as on their nameplate (manufacturer, type, hardware identifier), completed by the firmware version and the MSISDN of the SIM card, which can change during operation. Defined by LACS. The nameplate elements carry identification data only; configuration, firmware updates and operating values of the device are not part of LACS.

| Element | Meaning | Format |
| ------- | ------- | ------ |
| `npcmf` | Manufacturer of the controller | 1 to 32 characters, Unicode (UTF-8) |
| `npcty` | Type designation of the controller, including the hardware revision | 1 to 32 characters, Unicode (UTF-8) |
| `npcfw` | Firmware version of the controller | 1 to 32 characters, US-ASCII 32 to 126 |
| `npchi` | Hardware identifier of the controller, see below | 15 or 16 digits |
| `npcms` | MSISDN of the SIM card of the controller, see below | `+` followed by 1 to 15 digits (E.164) |
| `nplmf` | Manufacturer of the local unit | 1 to 32 characters, Unicode (UTF-8) |
| `nplty` | Type designation of the local unit, including the hardware revision | 1 to 32 characters, Unicode (UTF-8) |
| `nplfw` | Firmware version of the local unit | 1 to 32 characters, US-ASCII 32 to 126 |

`npchi`:

- If the controller has a mobile radio module, `npchi` is its IMEI with 15 digits (without software version, i.e. not the IMEISV).
- Otherwise `npchi` is the MAC address of the network interface over which the controller transmits LACS, as a decimal number plus 10^15. The result always has 16 digits and begins with `1`, e.g. MAC `00:1A:2B:3C:4D:5E` = 112394521950 → `1000112394521950`.
- The length distinguishes both cases. `npchi` is a string, because an IMEI can begin with `0`.

`npcms`:

- The MSISDN of the SIM card of the controller in E.164 format with `+` and without `tel:`, e.g. `+447700900123`. The LUC sends it only if the controller has a SIM card and knows its MSISDN.
- `npcms` is an identifier, not a medium: unlike `crd` it does not announce a way to reach the LUC. The ARC MUST NOT call the LUC at this number. The ARC MAY use it to associate calls from this number with the LUC, e.g. on a fallback alarm path over a mobile voice call, and to identify the SIM card.

Rules:

1. The LUC SHOULD include the nameplate elements in every test transmission (`stc=0102`). As every local unit sends its own test transmission, the ARC receives the data of the controller and of every local unit at least once per `automatic_test_interval`.
2. The LUC MAY include them in any other `mrq` with `mty=ME`.
3. The elements `npl…` describe the local unit denoted by `did` (and `iba`) of the message. Without `did` they are absent.
4. If controller and local unit are one device (`dty=0002`), the LUC MAY omit the elements `npl…`.
5. The LUC SHOULD send all nameplate elements it knows. All of them are optional.
6. The ARC MUST process a message independently of the nameplate elements. It MUST NOT reject or delay an alarm because of their content.
7. The ARC SHOULD store the data last received for each `cid` and `did` together with the time of receipt and SHOULD show it to the operator.

### custom extensions (x-...)

For flexible use, properties with the prefix "x-" can be defined without notice.

## Message Response (mrs)

### reference (ref)

as specified in TS 50134-9

### protocol-version (pve)

see [protocol-version (pve)](#protocol-version-pve) of the request

### status-number (snu)

| snu | Description | LUC behaviour for a voice request | Defined by |
| --- | ----------- | --------------------------------- | ---------- |
| 0 | OK, handled | Proceed; set up the voice call to `tnu` if `cre=62`. | TS 50134-9 |
| 1 | Message too long | As for `5`. | TS 50134-9 |
| 2 | Invalid format | As for `5`. | TS 50134-9 |
| 3 | Wrong data content (e.g. unknown `asid`, `did` and `iba` do not match, unknown `dco`) | As for `5`. | TS 50134-9 |
| 4 | Hold, the alarm is being processed | Repeat the request every 5 s with a new `ref` and the same `asid` until `snu=0` or until `alarm_accept_timeout` expires, then as for `5` ([Fallback Alarm Strategy](LACS-usecases.md#fallback-alarm-strategy-operator-queueing--failover)). | TS 50134-9 |
| 5 | Not treated or not distributed, try next alarm action | Reset (`RE`) and continue with the next server address, finally, for a lift voice alarm only, with a fallback alarm path, if the LUC has one. For `0202` the rules of the voice channel test apply; for `0211` and `0212` the session ends without `RE`. | TS 50134-9 |
| 6 | Busy, try again later or proceed with next alarm action | As for `4`, then as for `5`. | TS 50134-9 |
| 7 | Mandatory tag is missing | As for `5`. | TS 50134-9 |
| 10 | Reference missing | As for `5`. | TS 50134-9 |
| 11 | Not supported: the receiver does not support the requested action or function | Used by the LUC in responses to commands. For its own request: as for `5`. | LACS |
| 12 | Not permitted in current state: the receiver supports the action, but cannot carry it out in its current state | Used by the LUC in responses to commands, and by the ARC for a start of voice channel test during an open alarm; the LUC then postpones the test ([General rules](LACS-usecases.md#general-rules)). For any other own request: as for `5`. | LACS |
| 99 | Undefined error; also unknown message type (TS 50134-9 §7.2.2) and unsupported major version in `pve` (see [protocol-version (pve)](#protocol-version-pve)) | As for `5`; for an unsupported major version see [protocol-version (pve)](#protocol-version-pve), rule 4. | TS 50134-9 |

The column *LUC behaviour* is normative for voice requests (`stc` 0010, 0202, 0211, 0212). For a single event, any `snu` other than 4, 5 and 6 ends the repetition; on 4, 5 or 6 the LUC repeats it as specified in [Function monitoring and failure notification](LACS-usecases.md#function-monitoring-and-failure-notification), without `RE`. For responses to commands see [LACS-lacs-xml.md](LACS-lacs-xml.md#status-number-snu).

### status-text (ste)

as specified in TS 50134-9: printable US-ASCII, 1 to 128 characters, informative. Error responses use the texts `Error-unsupported-message`, `Error-unsupported-version`, `Error-unsupported-action`, `Error-not-permitted-in-state` and `Error-unknown-asid` as specified in this document and in [LACS-lacs-xml.md](LACS-lacs-xml.md#status-number-snu).

### media-reply (mre)

| mre | Description       | Defined by |
| --- | ----------------- | ---------- |
| 1   | duplex voice call | TS 50134-9 |

The ARC MUST send `mre=1` together with `cre=62`, i.e. whenever the LUC is to set up a voice call. Otherwise it MUST omit `mre`.

### callhandling-reply (cre)

| cre | Description                                                                                  | Defined by |
| --- | -------------------------------------------------------------------------------------------- | ---------- |
| 62  | set-up a voice call to transferred URI/Telephone number defined in transferred-number (`tnu`) | TS 50134-9 |

The first voice call immediately after the voice alarm (`stc`=0010) is always initiated by the LUC. Calls that the ARC requests by command (`aid` 0211, 0223) are also placed by the LUC. Within an alarm session the LUC MUST in addition accept a call of the ARC (see [Call Protection](LACS.md#call-protection)).

Only 62 is allowed here, so the ARC has full control regarding the call routing. If the call to `tnu` fails, the LUC continues according to the [Fallback Alarm Strategy](LACS-usecases.md#fallback-alarm-strategy-operator-queueing--failover).

### transferred-number (tnu)

as specified in TS 50134-9

### alert-session-id (asid)

The ARC MUST include the `asid` of the request in the response (see [alert-session-id (asid)](#alert-session-id-asid) of the request).

### custom extensions (x-...)

as in the [request](#custom-extensions-x-)

## Optional extension: Electronic lift attendant

This extension is OPTIONAL. A LUC or ARC that implements LACS for the alarm function only MAY ignore it.

In Germany, lifts have to be checked regularly for obvious defects and for their function by an appointed person, the lift attendant (BetrSichV Annex 1 No. 4.6, TRBS 3121 §3.3.2). These checks can be taken over partly or completely by an electronic lift attendant, i.e. a remote diagnostic system. Many lift alarm devices provide such a lift attendant module.

A LUC with an electronic lift attendant reports its events as single events (`mty=ME`, own `asid`, `dty` `0002` or `0045`). The codes and texts follow the events V001 to V010 of the "User Controlled Reporting" of the P100 protocol:

| stc  | Event                                          | Defined by |
| ---- | ---------------------------------------------- | ---------- |
| 0990 | V001 Lift attendant module active              | LACS       |
| 0991 | V002 Safety circuit fault                      | LACS       |
| 0992 | V003                                           | LACS       |
| 0993 | V004                                           | LACS       |
| 0994 | V005 Car light failure                         | LACS       |
| 0995 | V006 Number of unlevelled stops exceeded       | LACS       |
| 0996 | V007                                           | LACS       |
| 0997 | V008                                           | LACS       |
| 0998 | V009 Technician on site                        | LACS       |
| 0999 | V010 Lift attendant module back to normal operation | LACS       |

The meaning of V003, V004, V007 and V008 is not defined uniformly; it is manufacturer-specific.

- The events do not replace the alarm function and are not alarms in the sense of EN 81-28.
- An ARC that does not support this extension answers such messages with `snu=0` and MAY discard them, so that the LUC does not repeat them.
- The codes deliberately use the range for flexible use of TS 50134-9 (0900–0999), because they are locally limited. They are placed at the end of the range (0990–0999), so that collisions with codes defined from 0900 upwards are unlikely. They are not proposed for adoption into TS 50134-9.

## Request Overview

Column *Optional*: X = the LUC sends this message only if it can perform the respective check or provides the respective function.

| mty | dty                                                       | dco               | stc       | Optional |                                      |
| --- | --------------------------------------------------------- | ----------------- | --------- | -------- | ------------------------------------ |
| ME  | 0002 / 0100 / 0101 / <br />0102 / 0110                    |                   | 0010      |          | Lift voice alarm                     |
| ME  | 0002 / 0100 / 0101 / <br />0102 / 0110                    |                   | 0201      |          | End of Alarm                         |
| ME  | 0002 / 0041 /0045 / <br />0100 / 0101 / 0102 / <br />0110 |                   | 0014      |          | Battery failure / battery test failed |
| ME  | 0002 / 0041 /0045 / <br />0100 / 0101 / 0102 / <br />0110 |                   | 0210      | X        | Battery test passed                  |
| ME  | 0002 / 0041 /0045 / <br />0100 / 0101 / 0102 / <br />0110 |                   | 0015      | X        | Battery high/full/charged            |
| ME  | 0002 / 0041 /0045 / <br />0100 / 0101 / 0102 / <br />0110 |                   | 0016      | X        | Battery low                          |
| ME  | 0002 / 0041 /0045 / <br />0100 / 0101 / 0102 / <br />0110 |                   | 0085      | X        | Power failure (mains)                |
| ME  | 0002 / 0041 /0045 / <br />0100 / 0101 / 0102 / <br />0110 |                   | 0086      | X        | Power restored (mains)               |
| ME  | 0002 / 0045                                               |                   | 0078      | X        | Out of Service - emergency power supply depleted |
| ME  | 0002 / 0100 / 0101 / <br />0102 / 0110                    | 002–006           | 0204      | X        | Alarm button failure                 |
| ME  | 0002 / 0100 / 0101 / <br />0102 / 0110                    | 002–006           | 0208      | X        | Alarm button test passed             |
| ME  | 0002 / 0100 / 0101 / <br />0102 / 0110                    | 002–006 (Mic+Speaker) | 0205  | X        | Microphone speaker failure           |
| ME  | 0002 / 0100 / 0101 / <br />0102 / 0110                    | 002–006 (Mic+Speaker) | 0209  | X        | Microphone speaker test passed       |
| ME  | 0002 / 0045                                               |                   | 0023 / 0112 | X      | IP link failure / restored           |
| ME  | 0002 / 0045                                               |                   | 0024 / 0113 | X      | Mobile link failure / restored       |
| ME  | 0032 (Comm link monitor)                                  | -                 | 0206      | X        | Internal Bus failure                 |
| ME  | 0002 / 0045                                               |                   | 0202      |          | Start of voice channel test          |
| ME  | 0002 / 0045                                               |                   | 0203      |          | Voice channel failure                |
| ME  | 0002 / 0045                                               |                   | 0207      |          | Voice channel test passed            |
| ME  | 0002 / 0045 / 0100 / <br />0101 / 0102 / 0110             |                   | 0102      |          | Test transmission, primary channel   |
| ME  | 0002 / 0045 / 0100 / <br />0101 / 0102 / 0110             |                   | 0103      | X        | Test transmission, redundant channel |
| ME  | 0002 / 0100 / 0101 / <br />0102 / 0110                    |                   | 0211      |          | Voice call requested by the ARC (callback) |
| ME  | 0002 / 0100 / 0101 / <br />0102 / 0110                    |                   | 0212      | X        | Service voice call requested by the ARC |
| ME  | 0002 / 0045                                               |                   | 0990–0999 | X        | Electronic lift attendant (optional extension) |
| RE  | as in the request being reset                             | —                 | —         |          | Reset of a previously transmitted alarm session at this ARC (see [message-type (mty)](#message-type-mty) above) |
