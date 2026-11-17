# Changelog

All notable changes to the Lift Alarm Communication Standard (LACS) are documented here.

Entries marked **breaking** require existing implementations to be adapted. All other entries are unmarked.

As long as the version carries the suffix `-draft`, breaking changes are possible in every version. From 1.0.0-rc1 on, changes until 1.0.0 are limited to corrections that result from the review. From 1.0.0 on, the version has the form `MAJOR.MINOR.PATCH`; see *Versioning* in [LACS.md](LACS.md#versioning).

## 1.0.0-rc1 (2026-11-17)

Release candidate for 1.0.0. This section lists all changes since 0.12-draft, the last published version.

- **breaking** `stc` 0009 (alarm triggered by the system) removed, as no procedure is specified for it.
- The ARC calls the LUC at its `sip:` address-of-record; the LUC checks the server certificate against `sip_domain` (RFC 5922) and resolves DNS with A/AAAA only; the ARC uses symmetric RTP (RFC 4961).
- SIP error handling per response for REGISTER and MESSAGE; `Retry-After` is respected, except for requests of a lift voice alarm.
- Local unit hung up during HOLD or call setup: `RE`, the alarm stays open until it is initiated again; a failed callback within an alarm keeps the alarm session; a callback with an unknown `asid` belongs to an open alarm of the local unit.
- Voice channel test: every outcome other than a BYE with `cause=200` counts as failed; an alarm during a test does not wait for the response to the reset.
- An alarm during a call outside an alarm: the LUC ends a service voice call, keeps a callback of the same local unit (recommended); `0211` and `0223` are answered with `snu=12` while the voice channel is busy.
- A voice channel test postponed for longer than `automatic_test_interval` is reported as failed (`0203`); the ARC does not hold voice channel tests, single events, callbacks and service voice calls.
- `pve`: a message without `pve` counts as `LACS/1.0`; the schema accepts other values, which are answered with `snu=99`.
- Keep-alive at most every 120 s without `Flow-Timer`; re-registration after a random delay of up to 30 s, at once during an alarm.
- The CA certificate has no expiry either; a LUC without a set clock does not reject certificates because of their validity; the ARC includes PCMA in every offer.
- `LACS-ASID` mandatory; `stc` mandatory in messages; video capabilities `0022`, `0023` experimental; attention tones also for an accepted incoming call outside an alarm.
- The local indication of a failed test transmission also ends when a voice request is answered with `snu=0`.
- An alarm while the voice channel is busy or during a callback or service voice call outside an alarm: the LUC sends the voice request when the running call has ended.
- `cap` in every voice request and test transmission; `cap` 0015 real-time text (experimental); `stt` with the reason of a failed call.
- Callback with an unknown `asid` is carried out; attention tones throughout a callback that the LUC cannot assign to an alarm.
- Voice channel test: at most 2 further test runs after 30 s to 120 s, a fallback alarm path is used for a lift voice alarm only; the ARC reports a missing test transmission after `automatic_test_interval` plus 1 h.
- Internal systems behind a TLS-terminating front system of the ARC accept requests from the front system only.
- Operational certificates: `notAfter` `99991231235959Z`; XML schema checks conditional elements with assertions.
- **breaking** `did` consists of letters, digits, `-`, `.`, `_` and `~` only (token according to RFC 3261), so that it can be carried unquoted in SIP headers.
- Every `mrq` and `mrs` is sent as one SIP MESSAGE; `ame` (splitting at 1300 bytes) is not used; body at most 4096 bytes except the snapshot response.
- **breaking** Authentication by TLS client certificate (CN = `cid`); SIP Digest is optional.
- **breaking** SIP profile: REGISTER over a persistent TLS connection with CRLF keep-alive, SIP Outbound (RFC 5626) recommended, TCP with TLS only, no SIP INFO and no DTMF codes (TS 50134-9 clause 8); media profile with G.711 A-law, G.729 (mandatory for the ARC, recommended for the LUC) and SRTP `AES_CM_128_HMAC_SHA1_80`.
- **breaking** The `asid` of a call travels in the SIP header `LACS-ASID` instead of the Contact parameter.
- **breaking** Callback: the ARC requests it with `aid=0211`, the LUC sends a voice request (`stc` 0010 or 0211) and places the call; within an alarm session the ARC may still call the LUC directly. Call protection is based on the alarm session and the TLS connection of the registration instead of the `asid`.
- **breaking** `application/lacs+xml` is mandatory for LUC and ARC, with the commands `aid` 0210 to 0214.
- **breaking** Voice channel test: request and response carry `mreq=1`/`mre=1`; the LUC always reports the result (`stc` 0207 passed, 0203 failed) and the test ends with this report.
- **breaking** "Help is on the way" is the command `aid=0214` (replaces `TD` `0131`).
- **breaking** Device components: `002`–`006` "Alarm Button / Voice Unit 1–5" replace `090`, camera `030` and TBUI `031` replace `091` and `092`.
- **breaking** Internal bus address `iba` is mandatory if a controller has more than one local unit.
- New status codes: test passed `0207`–`0210`, out of service `0078`, voice request on command `0211`/`0212`; `ico` `001`/`002` for alarm filtering.
- New commands: `0213` voice channel test, `0215`–`0219`, `0221`, `0222` audio control, `0223` service voice call, `0224` select voice unit; up to five voice units per local unit.
- Status numbers `11` and `12` for responses to commands; the ARC evaluates every response. The response only confirms receipt (never `snu=4`); the LUC sends the effect message within 60 s.
- General rules: one lift voice alarm per local unit, the ARC accepts every alarm; a voice channel test runs only without an open alarm (`aid=0213` and `stc=0202` are answered with `snu=12`), an alarm ends a running test; the result report follows within 30 s.
- XML schema: `mreq` and `mre` only `1`, `cre` only `62`.
- Every local unit sends its own test transmission; `did` mandatory for messages of a local unit; `ref` unique for 600 s.
- Reference to prEN 81-28:2025 (draft) instead of EN 81-28:2022.
- Acknowledgement (`snu=0` to the voice request) and green symbol while the voice call is established; voice call after acceptance fails; repetition across all targets, alternative ARC within 5 min (`alarm_accept_timeout` at most 240 s).
- Configuration: `alarm_uri`, `automatic_test_uri`, `automatic_test_interval`, `ca_certificate`, client certificate and key, `call_protection`; `alarm_fallback_channel` removed.
- `asid`: UUID according to RFC 9562 (versions 1 to 8); sessions opened by commands of the ARC.
- XML schemas: XML Schema 1.1 without namespace, unknown elements accepted, codes checked by syntax only.
- Optional extension *Electronic lift attendant* (`stc` 0990–0999).
- TBUI: free text and dynamic answers are experimental.
- New sections *Conventions* (BCP 14), *Timers*, *Security Considerations*, *Network Requirements* and *Conformance*; deliberate deviations from TS 50134-9 marked; use cases with normative procedures.
- Protocol name and version `pve` (e.g. `LACS/1.0`) in every `mrq` and `mrs`, *Versioning*; `ver` and `cve` of TS 50134-9 are not used.
- Event time `etm` (RFC 3339, UTC) in messages of the LUC; `tim` within `lge` is not used.
- Nameplate elements `npcmf`, `npcty`, `npcfw`, `npchi`, `npcms`, `nplmf`, `nplty`, `nplfw` (manufacturer, type, firmware, hardware identifier and MSISDN of controller and local unit) in the test transmission.
- Message exchange: repetition with the same `ref` if no response arrives within 5 s, detection of repeated requests, late responses are discarded.
- Single events are kept until they are answered; random variation of backoff times and of the time of the test transmission.
- Supervision of the alarm path by the registration (expiry at most 1800 s); TLS 1.3 and AEAD cipher suites recommended; SRTP `AEAD_AES_128_GCM` optional.
- Examples use the values of one sample installation instead of placeholders.
- New informative document `FAQ.md` with the reasons for selected design decisions.
- Version in the form `MAJOR.MINOR.PATCH`; `pve` carries `LACS/MAJOR.MINOR`; experimental functions are excluded from the compatibility rules.
- Behaviour of the LUC for every status number of a voice request; `ste` restricted to printable US-ASCII as in TS 50134-9.
- Commands: the ARC may repeat an unanswered command; the response to the camera snapshot carries the image and follows within 10 s.
- Reset (`RE`) carries the `ref` of the first request to this ARC since the last reset there; a voice channel test ended by an alarm reports no result.
- Test transmission: successful with `snu=0`; local indication of a failed test transmission after 1 h; `automatic_test_interval` at most 72 h.
- Request overview aligned with *Conformance*; data minimisation does not apply to voice, camera images and TBUI text.

## 0.12-draft (2026-05-13)

The entries for 0.12-draft and earlier are reconstructed from the commit history. Breaking changes are not marked.

- Use case *Fallback Alarm Strategy*: HOLD with retransmission, reset (`mty=RE`) and re-routing to the next ARC.
- `mty=RE` (TS 50134-9 §5.4) with the `asid` of the session; the ARC answers `snu=0`, also for an unknown `asid`.
- New configuration parameters `alarm_accept_timeout` (default 180 s) and `alarm_fallback_channel`.

## 0.11-draft (2026-05-12)

- RFC 3326 referenced for the `Reason` header of the BYE in the voice channel test.

## 0.10-draft (2026-05-12)

- Voice channel test inverted: the ARC sends the test tone, the LUC echoes it, the ARC evaluates.
- Test result in the `Reason` header of the BYE (`cause` 200/603) with example texts; the LUC reports failures with `stc` 0203.

## 0.09-draft (2026-04-22)

- Specification split into `LACS.md`, `LACS-scaip-xml.md`, `LACS-lacs-xml.md` and `LACS-usecases.md`.

## 0.08-draft (2026-04-22)

- New section *Call Protection*.
- `asid` stable only during lift alarm and voice channel test; the ARC creates the `asid` for commands outside an alarm.
- A controller can serve several local units.
- First XML schema drafts for `scaip+xml` and `lacs+xml`.

## 0.07-draft (2026-02-27)

- New section *Normative References*.
- LUC configuration as a table; `sip_domain`, `sip_outbound_proxy`, `sip_port` and `pki_ca_root_certificate`.

## 0.06-draft (2026-02-11)

- LACS status codes moved from the 09xx block to the 02xx block.
- SIP voice channel test with `stc` 0202 replaces the echo test.
- `cap` values with four digits (e.g. `0011` instead of `11`).
- New commands `aid` 0210 (test transmission), 0211 (voice callback) and 0212 (end of alarm).
- Custom extensions with the prefix `x-`.
- UDP no longer used for SIP MESSAGE.
- "Heartbeat" replaced by the test transmission.

## 0.05-draft (2026-01-23)

- LACS status codes for failures of alarm button, microphone/speaker and internal bus.

## 0.04-draft (2026-01-22)

- Periodic test transmission with `mty=ME` instead of `PI`, as in the ELA Information Note.
- Section *Additional Permissions for Standardisation Bodies*.

## 0.03 (2025-11-27)

- New section *Motivation*.
- Corrected MESSAGE responses in the sequence diagram for text-based alarm communication.

## 0.02-draft (2025-11-27)

- First published version.
