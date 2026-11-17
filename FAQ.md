# LACS — Frequently Asked Questions

**Part of:** [Lift Alarm Communication Standard (LACS)](LACS.md), see there for version and status

This document is informative. It explains selected design decisions of LACS, in particular where LACS deliberately deviates from CLC/TS 50134-9 or from common expectations. If it contradicts the specification, the specification applies.

## Protocol design

### Why does LACS use numeric codes (e.g. `dty`, `stc`) instead of readable text?

LACS is based on TS 50134-9, which defines numeric codes for device types, status codes and components and two-letter codes for message types. LACS keeps these codes, so that a receiver that implements TS 50134-9 understands LACS messages, and adds its own codes in ranges that TS 50134-9 leaves free. Texts such as `ite` are informative only.

### Why is SIP carried over TCP with TLS only?

TLS protects the signalling against eavesdropping and manipulation and authenticates both sides by certificates. TCP also carries large messages, e.g. a camera snapshot of up to 1 MiB, without IP fragmentation, which is unreliable over UDP in many networks.

### Why is there no globally unique ID for controllers and local units?

LACS identifies a controller by its `cid` and a local unit by `cid` and `did`, as TS 50134-9 does. The `cid` comes from the numbering of the ARC: alarms end up in the existing alarm management systems of the ARCs, each with its own numbering scheme. LACS cannot assume which identifier formats these systems accept, nor assign values that are free of collisions in every ARC. Where the hardware has to be recognised across ARCs, the hardware identifier `npchi` (IMEI or MAC address of the controller) serves this purpose.

### Why does LACS have no heartbeat (`PI`)?

TS 50134-9 §5.5 provides a periodic heartbeat message from the alarm sender to the alarm receiver; LACS does not use it. The LUC has to be registered anyway: the ARC reaches it with commands and calls, and as the LUC is usually behind NAT, only over the persistent TLS connection that the LUC keeps open with CRLF keep-alive messages (RFC 5626). If the registration expires (at most 1800 s are recommended), the ARC treats this as a failure of the alarm path, so a failure is detected within about 30 min. A heartbeat would report the same a second time. The automatic test required by EN 81-28 at least every 3 days is covered by the test transmission and the voice channel test.

### Why does LACS use `pve` instead of `ver` and `cve` of TS 50134-9?

`ver` and `cve` carry the version of TS 50134-9. What matters for interoperability is the LACS version, because it determines which elements, codes and commands a side supports. `pve` carries the protocol name and `MAJOR.MINOR` of LACS (e.g. `LACS/1.0`) in every request and response, so each side knows that the other side speaks LACS and which functions it supports. The edition of TS 50134-9 used by LACS follows from the LACS version.

### Why is SIP INFO not used?

All control of the LUC by the ARC uses `lacs+xml` commands with `ref`, `asid` and a response with a status number. Every action is thus confirmed and traceable, also through session border controllers that may drop or alter SIP INFO. For the same reason LACS does not use the DTMF codes of TS 50134-9 clause 8.

### Why does a repetition after HOLD (`snu=4`) get a new `ref`?

TS 50134-9 §5.2 requires the same message request to be resent. In LACS every request remains individually traceable by its own `ref`, and the unchanged `asid` tells the ARC that it is a repetition of the same alarm. A request that has not been answered at all is repeated with the same `ref`, so that the ARC can detect it as a duplicate.

### Why does `pve` carry only `MAJOR.MINOR` of the version?

The version of LACS has the form `MAJOR.MINOR.PATCH`. A patch version only corrects or clarifies the documents and changes nothing on the wire, so the other side does not need to know it. All release candidates of a version send the same `pve` as the final version.

### Why does LACS 1.0 refer to the draft prEN 81-28:2025?

The requirements of the coming edition of EN 81-28, e.g. on acknowledgement, alarm filtering and the automatic test, are known from the draft and agreed with the European Lift Association. LACS follows them now, so that devices do not have to be changed when the edition is published. LACS will refer to the published edition in a later version.

## Security

### Why client certificates instead of HTTP Digest authentication?

Passwords can leak and cannot be revoked centrally, and TS 50134-9 even allows group accounts. A certificate per device, with the `cid` as its subject, authenticates the LUC already on the TLS connection, and the ARC withdraws a device by revoking its authorisation. SIP Digest remains optional in addition; this deliberately deviates from TS 50134-9 §4.2.2.

### Why does call protection not depend on the `asid`?

Session border controllers and other intermediate SIP systems can remove unknown headers such as `LACS-ASID`. Call protection therefore decides by the state of the LUC (an alarm session is active) and by the connection (the call arrives over the TLS connection of its registration). Calls requested by the ARC outside an alarm are placed by the LUC itself and are not affected.

## Voice channel test

### Why does the ARC send the test signal and the LUC only echo it?

The LUC only loops the received audio back. The signal analysis runs in the ARC, so that the voice channel is assessed in the same way for devices of all manufacturers and the LUC stays simple.

### Why does the LUC report the result, although the ARC evaluates it?

In many ARCs the media endpoint that evaluates the echo is not the component that is connected to the alarm management system. Routing the result back through the LUC (`stc` 0207 or 0203) brings it to the ARC on the same path as the results of all other periodic tests.

## Event time and nameplate

### Why is the event time (`etm`) always in UTC, without a time zone?

The ARC knows the address of every lift and thus its time zone, including daylight saving time. A time zone configured in the device could contradict it, e.g. after a wrong setting, and an offset such as `+01:00` carries no daylight saving rules. With UTC there is one source of truth, and the LUC only needs a synchronised UTC clock. How the LUC shows local time on its own display is up to the device.

### Why does LACS not use `tim` within `lge` for the event time?

In TS 50134-9, `tim` is a sub-element of `lge` (location-geo), i.e. a time stamp that belongs to a geographical position, not the time of the reported event. Its time zone offset allows whole hours only. A separate element `etm` states the meaning unambiguously and uses RFC 3339 in UTC. Receivers ignore `lge/tim`.

### Why is the MSISDN sent in `npcms` and not as a further `crd`?

In TS 50134-9, `crd` names the media of the alarm sender, which the ARC may use to reach it, e.g. for a callback. A lift alarm device usually must not be called on the number of its SIM card. `npcms` is an identifier without `tel:`: the ARC does not call it, but it can use it to associate calls from this number, e.g. on a fallback alarm path, and to identify the SIM card.
