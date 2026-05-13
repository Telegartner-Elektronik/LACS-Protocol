# LACS — Use Cases and Sequence Diagrams

**Part of:** [Lift Alarm Communication Standard (LACS)](LACS.md) v0.08-draft

> [!WARNING]
>
> All sequence diagrams below are rough drafts and may still contain serious errors.

For message property definitions, see:
- [LACS-scaip-xml.md](LACS-scaip-xml.md) — `application/scaip+xml` message properties
- [LACS-lacs-xml.md](LACS-lacs-xml.md) — `application/lacs+xml` message properties

---

## scaip+xml Use Cases

The following use cases use `Content-Type: application/scaip+xml` for signalling.

### Lift Voice Alarm

```mermaid
sequenceDiagram
    participant LUC
    participant ARC

    Note over LUC: User presses alarm button<br>START OF ALARM

    LUC->>ARC: (1) MESSAGE Request - Lift voice alarm
    Note right of ARC: Alarm identification and<br> location data received

    ARC->>LUC: (2) MESSAGE Response - Lift voice alarm
    Note left of LUC: Acknowledgement received.<br>Proceed to voice setup.

    LUC->>ARC: (3) INVITE - initial call
    ARC->>LUC: 200 OK

	Note over LUC, ARC: Two-way speech established

    ARC->>LUC: (4) BYE - initial call
    LUC->>ARC: 200 OK
    Note over LUC: Voice session terminated

    opt optional callback from ARC
        ARC->>LUC: (5) INVITE - callback
        LUC->>ARC: 200 OK

        Note over LUC, ARC: Two-way speech established

        ARC->>LUC: (6) BYE - callback
        LUC->>ARC: 200 OK
        Note over LUC: Voice session terminated
    end

    Note left of LUC: Wait for rescue...
    Note over LUC: END OF ALARM is<br> triggered at device

    LUC->>ARC: (7) MESSAGE Request - End of alarm
    Note right of ARC: Signalling termination<br> of the entrapment state

    ARC->>LUC: (8) MESSAGE Response - End of alarm
    Note over ARC: Alarm handling successfully concluded
```

#### (1) MESSAGE Request - Lift voice alarm

```xml
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrq>
    <ref>{ref-1}</ref>
    <mty>ME</mty>
    <mreq>1</mreq>
    <cid>{controller-id}</cid>
    <dty>0101</dty>
    <did>{device-id}</did>
    <stc>0010</stc>
    <lco>053</lco>
    <asid>{alert-session-id}</asid>
</mrq>
```

#### (2) MESSAGE Response - Lift voice alarm

```xml
MESSAGE sip:{controller-id}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrs>
	<ref>{ref-1}</ref>
    <snu>0</snu>
    <mre>1</mre>
    <cre>62</cre>
    <tnu>sips:{arc-operator}@domain.tld</tnu>
    <asid>{alert-session-id}</asid>
</mrs>
```

#### (3) INVITE - initial call

```
INVITE sip:{arc-operator}@domain.tld;transport=tls

From: <sip:{controller-id}@domain.tld>;did={device-id}
To: <sip:{arc-operator}@domain.tld>
Contact: <sip:{controller-id}@{ip}:{port};transport=tls;asid={alert-session-id}>
...
```

#### (4) BYE - initial call

```
BYE sip:{controller-id}@{ip}:{port};transport=tls;asid={alert-session-id}
...
```

#### (5) INVITE - callback

```
INVITE sip:{controller-id}@domain.tld;transport=tls;did={device-id}

From: <sip:{arc-operator}@domain.tld>;did={device-id}
To: <sip:{controller-id}@domain.tld>
Contact: <sip:{arc-operator}@{ip}:{port};transport=tls;asid={alert-session-id}>
...
```

#### (6) BYE - callback

```
BYE sip:{controller-id}@{ip}:{port};transport=tls;asid={alert-session-id}
...
```

#### (7) MESSAGE Request - End of alarm

```xml
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrq>
    <ref>{ref-2}</ref>
    <mty>ME</mty>
    <cid>{controller-id}</cid>
    <dty>0101</dty>
    <did>{device-id}</did>
    <stc>0201</stc>
    <lco>053</lco>
    <asid>{alert-session-id}</asid>
</mrq>
```

#### (8) MESSAGE Response - End of alarm

```xml
MESSAGE sip:{controller-id}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrs>
	<ref>{ref-2}</ref>
    <snu>0</snu>
    <asid>{alert-session-id}</asid>
</mrs>
```

### Fallback Alarm Strategy (Operator Queueing & Failover)

While all operators are busy, the ARC replies with `snu=4` (HOLD) and the LUC retransmits the alarm with a new `ref` and the same `asid`. Once an operator becomes available, the ARC answers with `snu=0` and the operator's URI in `tnu`.

If `alarm_accept_timeout` elapses without `snu=0`, the LUC aborts the alarm at this ARC with `mty=RE` (TS 50134-9 §5.4) and re-routes it to the next entry of `alarm_uri_list`, keeping the same `asid` (same alarm case).

If every ARC in `alarm_uri_list` has been exhausted, the LUC falls back to the configured `alarm_fallback_channel` (see [LACS.md](LACS.md)).

```mermaid
sequenceDiagram
    participant LUC
    participant ARC1 as primary ARC
    participant ARC2 as alternative ARC
    participant OOB as alarm_fallback_channel<br>(voice call / SMS / HTTP)

    Note over LUC: User presses alarm button<br>START OF ALARM

    LUC->>ARC1: (1) MESSAGE mrq - Lift voice alarm<br>ref-1, asid-A
    Note right of ARC1: All operators busy<br>→ queue alarm
    ARC1->>LUC: (2) MESSAGE mrs - HOLD (snu=4)<br>ref-1, asid-A

    Note over LUC: Wait 5 s, then retransmit<br>(new ref, same asid)

    LUC->>ARC1: (3) MESSAGE mrq - Lift voice alarm<br>ref-2, asid-A
    ARC1->>LUC: (4) MESSAGE mrs - HOLD (snu=4)<br>ref-2, asid-A

    Note over LUC: ... further retries while<br>operators remain busy ...

    alt operator on ARC1 becomes available before alarm_accept_timeout
        LUC->>ARC1: (5) MESSAGE mrq - Lift voice alarm<br>ref-n, asid-A
        Note right of ARC1: Operator X picks up alarm
        ARC1->>LUC: (6) MESSAGE mrs - OK (snu=0)<br>tnu=sips:{arc-operator-x}@...<br>ref-n, asid-A

        LUC->>ARC1: (7) INVITE sip:{arc-operator-x}@...
        ARC1->>LUC: 200 OK
        Note over LUC, ARC1: Two-way speech established
        ARC1->>LUC: BYE
        LUC->>ARC1: 200 OK

        Note over LUC: continue with regular<br>alarm flow / EOA
    else alarm_accept_timeout on ARC1 expires (still snu=4)
        Note over LUC: Stop retrying on ARC1<br>→ abort there + re-route to ARC2<br>(same asid — same alarm case)

        LUC->>ARC1: (8) MESSAGE mrq - Reset<br>mty=RE, ref=ref-1, asid-A
        ARC1->>LUC: (9) MESSAGE mrs - OK (snu=0)<br>ref=ref-1, asid-A
        Note right of ARC1: Remove alarm from queue

        LUC->>ARC2: (10) MESSAGE mrq - Lift voice alarm<br>ref-m, asid-A

        alt operator on ARC2 becomes available before alarm_accept_timeout
            ARC2->>LUC: (11) MESSAGE mrs - OK (snu=0)<br>tnu=sips:{arc2-operator}@...<br>ref-m, asid-A

            LUC->>ARC2: (12) INVITE sip:{arc2-operator}@...
            ARC2->>LUC: 200 OK
            Note over LUC, ARC2: Two-way speech established
            ARC2->>LUC: BYE
            LUC->>ARC2: 200 OK

            Note over LUC: continue with regular<br>alarm flow / EOA
        else alarm_accept_timeout on ARC2 also expires
            Note over LUC: All ARCs in alarm_uri_list exhausted<br>→ fallback channel

            LUC->>ARC2: (13) MESSAGE mrq - Reset<br>mty=RE, ref=ref-m, asid-A
            ARC2->>LUC: (14) MESSAGE mrs - OK (snu=0)<br>ref=ref-m, asid-A

            LUC->>OOB: (15) voice call / SMS / HTTP webhook<br>(asid-A in payload if supported)
            Note over OOB: Operator handles alarm<br>out-of-band
        end
    end
```

> [!NOTE]
>
> The LUC retransmits after `snu=4` every **5 s** (lower bound of the 5–20 s range in TS 50134-9 §5.2). Step (15) is outside the LACS message flow.

#### (1) MESSAGE Request - Lift voice alarm (initial attempt)

```xml
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrq>
    <ref>{ref-1}</ref>
    <mty>ME</mty>
    <mreq>1</mreq>
    <cid>{controller-id}</cid>
    <dty>0101</dty>
    <did>{device-id}</did>
    <stc>0010</stc>
    <lco>053</lco>
    <asid>{alert-session-id}</asid>
</mrq>
```

#### (2) MESSAGE Response - HOLD (all operators busy)

```xml
MESSAGE sip:{controller-id}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrs>
    <ref>{ref-1}</ref>
    <snu>4</snu>
    <asid>{alert-session-id}</asid>
</mrs>
```

#### (3) MESSAGE Request - Lift voice alarm (retry)

```xml
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrq>
    <ref>{ref-2}</ref>
    <mty>ME</mty>
    <mreq>1</mreq>
    <cid>{controller-id}</cid>
    <dty>0101</dty>
    <did>{device-id}</did>
    <stc>0010</stc>
    <lco>053</lco>
    <asid>{alert-session-id}</asid>
</mrq>
```

#### (4) MESSAGE Response - still HOLD

```xml
MESSAGE sip:{controller-id}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrs>
    <ref>{ref-2}</ref>
    <snu>4</snu>
    <asid>{alert-session-id}</asid>
</mrs>
```

#### (5) MESSAGE Request - Lift voice alarm (further retry)

```xml
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrq>
    <ref>{ref-n}</ref>
    <mty>ME</mty>
    <mreq>1</mreq>
    <cid>{controller-id}</cid>
    <dty>0101</dty>
    <did>{device-id}</did>
    <stc>0010</stc>
    <lco>053</lco>
    <asid>{alert-session-id}</asid>
</mrq>
```

#### (6) MESSAGE Response - Operator available (snu=0)

```xml
MESSAGE sip:{controller-id}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrs>
    <ref>{ref-n}</ref>
    <snu>0</snu>
    <mre>1</mre>
    <cre>62</cre>
    <tnu>sips:{arc-operator-x}@domain.tld</tnu>
    <asid>{alert-session-id}</asid>
</mrs>
```

#### (7) INVITE - initial call to the assigned operator

```
INVITE sip:{arc-operator-x}@domain.tld;transport=tls

From: <sip:{controller-id}@domain.tld>;did={device-id}
To: <sip:{arc-operator-x}@domain.tld>
Contact: <sip:{controller-id}@{ip}:{port};transport=tls;asid={alert-session-id}>
...
```

From here on the flow continues as in the regular *Lift Voice Alarm* use case.

#### (8) MESSAGE Request - Reset of alarm at primary ARC

```xml
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrq>
    <ref>{ref-1}</ref>
    <mty>RE</mty>
    <cid>{controller-id}</cid>
    <did>{device-id}</did>
    <asid>{alert-session-id}</asid>
</mrq>
```

#### (9) MESSAGE Response - Reset acknowledged by primary ARC

```xml
MESSAGE sip:{controller-id}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrs>
    <ref>{ref-1}</ref>
    <snu>0</snu>
    <asid>{alert-session-id}</asid>
</mrs>
```

#### (10) MESSAGE Request - Lift voice alarm to alternative ARC

```xml
MESSAGE sip:{arc-reception-alt}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrq>
    <ref>{ref-m}</ref>
    <mty>ME</mty>
    <mreq>1</mreq>
    <cid>{controller-id}</cid>
    <dty>0101</dty>
    <did>{device-id}</did>
    <stc>0010</stc>
    <lco>053</lco>
    <asid>{alert-session-id}</asid>
</mrq>
```

#### (11) MESSAGE Response - OK from alternative ARC

```xml
MESSAGE sip:{controller-id}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrs>
    <ref>{ref-m}</ref>
    <snu>0</snu>
    <mre>1</mre>
    <cre>62</cre>
    <tnu>sips:{arc2-operator}@domain.tld</tnu>
    <asid>{alert-session-id}</asid>
</mrs>
```

#### (12) INVITE - initial call to the alternative ARC operator

```
INVITE sip:{arc2-operator}@domain.tld;transport=tls

From: <sip:{controller-id}@domain.tld>;did={device-id}
To: <sip:{arc2-operator}@domain.tld>
Contact: <sip:{controller-id}@{ip}:{port};transport=tls;asid={alert-session-id}>
...
```

#### (13) MESSAGE Request - Reset of alarm at alternative ARC

```xml
MESSAGE sip:{arc-reception-alt}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrq>
    <ref>{ref-m}</ref>
    <mty>RE</mty>
    <cid>{controller-id}</cid>
    <did>{device-id}</did>
    <asid>{alert-session-id}</asid>
</mrq>
```

#### (14) MESSAGE Response - Reset acknowledged by alternative ARC

```xml
MESSAGE sip:{controller-id}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrs>
    <ref>{ref-m}</ref>
    <snu>0</snu>
    <asid>{alert-session-id}</asid>
</mrs>
```

#### (15) Fallback channel (out-of-band)

Out-of-band transmission over `alarm_fallback_channel` (see [LACS.md](LACS.md)). Carrier and payload are implementation-defined.

### Test Transmission

```mermaid
sequenceDiagram
    participant LUC
    participant ARC

    Note over LUC: Internal Timer (test_interval) expires

    LUC->>ARC: (1) MESSAGE Request - Test Transmission
    Note right of ARC: ARC verifies identification and<br>processes message<br>(Test transmission)

    ARC->>LUC: (2) MESSAGE Response - Test Transmission


    Note over LUC: If Status 0=OK, reset the internal timer.<br>If test fails, activate alternate blinking pictograms.

```

#### (1) MESSAGE Request - Test Transmission

```xml
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrq>
    <ref>{ref-1}</ref>
    <mty>ME</mty>
    <cid>{controller-id}</cid>
    <dty>0101</dty>
    <did>{device-id}</did>
    <stc>0102</stc>
    <lco>053</lco>
    <asid>{alert-session-id}</asid>
</mrq>
```

#### (2) MESSAGE Response - Test Transmission

```xml
MESSAGE sip:{controller-id}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrs>
	<ref>{ref-1}</ref>
    <snu>0</snu>
    <asid>{alert-session-id}</asid>
</mrs>
```

### Power Outage

#### Short Power Outage

A short outage results in a **failure and a restoration message**.

```mermaid
sequenceDiagram
    participant LUC
    participant ARC

    Note over LUC: Loss of primary AC power supply
    LUC->>ARC: (1) MESSAGE mrq Power failure
    ARC->>LUC: (2) MESSAGE mrs OK

    Note over LUC: System runs on emergency backup power

    Note over LUC: Primary AC power restored
    LUC->>ARC: (3) MESSAGE mrq Power restored
    ARC->>LUC: (4) MESSAGE mrs OK

```

##### (1) MESSAGE mrq — Power failure

```xml
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrq>
    <ref>{ref-1}</ref>
    <mty>ME</mty>
    <cid>{controller-id}</cid>
    <dty>0045</dty>
    <stc>0085</stc>
    <asid>{alert-session-id-1}</asid>
</mrq>
```

##### (2) MESSAGE mrs — Power failure OK

```xml
MESSAGE sip:{controller-id}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrs>
    <ref>{ref-1}</ref>
    <snu>0</snu>
    <asid>{alert-session-id-1}</asid>
</mrs>
```

##### (3) MESSAGE mrq — Power restored

```xml
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrq>
    <ref>{ref-2}</ref>
    <mty>ME</mty>
    <cid>{controller-id}</cid>
    <dty>0045</dty>
    <stc>0086</stc>
    <asid>{alert-session-id-2}</asid>
</mrq>
```

##### (4) MESSAGE mrs — Power restored OK

```xml
MESSAGE sip:{controller-id}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrs>
    <ref>{ref-2}</ref>
    <snu>0</snu>
    <asid>{alert-session-id-2}</asid>
</mrs>
```

#### Long Power Outage

If the mains is not restored, the emergency power supply may become critically low.
In this case **no "power restored" message is sent**, and a **battery low** status must be transmitted before shutdown.

```mermaid
sequenceDiagram
    participant LUC
    participant ARC

    Note over LUC: Loss of primary AC power supply
    LUC->>ARC: MESSAGE mrq Power failure
    ARC->>LUC: MESSAGE mrs OK

    Note over LUC: System runs on emergency backup power...

    Note over LUC: Emergency battery capacity low
    LUC->>ARC: (1) MESSAGE mrq Battery low
    ARC->>LUC: (2) MESSAGE mrs OK

    Note over LUC: System shutdown until mains power returns

```

##### (1) MESSAGE mrq — Battery low

```xml
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrq>
    <ref>{ref-3}</ref>
    <mty>ME</mty>
    <cid>{controller-id}</cid>
    <dty>0045</dty>
    <stc>0016</stc>
    <asid>{alert-session-id-2}</asid>
</mrq>
```

##### (2) MESSAGE mrs — Battery low ACK

```xml
MESSAGE sip:{controller-id}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrs>
    <ref>{ref-3}</ref>
    <snu>0</snu>
    <asid>{alert-session-id-2}</asid>
</mrs>
```

### Function monitoring and failure notification

LUC should check at least on every `test_interval`:

| To be checked                                                                                                  | Comment                                                                                            | mrq on failure                                                      |
| -------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| Condition emergency supply for local unit(s) and controller is good and it fulfils the requirements of EN81-28 | Mandatory according to EN81-28                                                                     | Battery failure<br />`stc` = 0014                                   |
| Voice channel (RTP) towards ARC works properly                                                                 | Must be checked using voice channel test sequence (see section below)                              | Voice channel (RTP) failure<br />`stc` = 0203                       |
| Pressing and releasing the alarm button at local unit works properly                                           | Optional - Only possible with special hardware such as a remotely operated button                  | Alarm button failure<br />`dco` = 002 / 003 / 004<br />`stc` = 0204 |
| Proper functionality of microphone and loudspeaker at local unit                                               | Optional - Audio signal played through speaker must be received by microphone at appropriate level | Microphone speaker failure<br />`dco` = 090<br />`stc` = 0205       |
| Communication link of internal bus between Local Unit(s) and Controller is available                           | Optional                                                                                           | Internal Bus failure<br />`dty` = 0032<br />`stc` = 0206            |

If a failure is detected:

1. The LUC must immediately transmit a `mrq` `MESSAGE` with the appropriate properties.
2. If the failure still persists on next check a new message must be send with the same failure. The interval at which the LUC performs function checks must not exceed the configured `test_interval`.

#### Generic Sequence diagram

```mermaid
sequenceDiagram
    participant LUC
    participant ARC

    Note over LUC: Internal monitoring detects a failure

    LUC->>ARC: (1) MESSAGE mrq Failure X
    ARC->>LUC: (2) MESSAGE mrs OK

    opt failure persists on next test interval?
        LUC->>ARC: (3) MESSAGE mrq Failure X
        ARC->>LUC: (4) MESSAGE mrs OK
    end
```

---

#### Example: Battery Failure (0014)

##### (1) MESSAGE mrq - Battery failure

```xml
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrq>
    <ref>{ref-1}</ref>
    <mty>ME</mty>
    <cid>{controller-id}</cid>
    <dty>0045</dty>
    <stc>0014</stc>
    <asid>{alert-session-id-1}</asid>
</mrq>
```

##### (2) MESSAGE mrs OK

```xml
MESSAGE sip:{controller-id}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrs>
    <ref>{ref-1}</ref>
    <snu>0</snu>
    <asid>{alert-session-id-1}</asid>
</mrs>
```

The failure persists on next test interval.

##### (3) MESSAGE mrq - Battery failure

```xml
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrq>
    <ref>{ref-2}</ref>
    <mty>ME</mty>
    <cid>{controller-id}</cid>
    <dty>0045</dty>
    <stc>0014</stc>
    <asid>{alert-session-id-2}</asid>
</mrq>
```

##### (4) MESSAGE mrs OK

```xml
MESSAGE sip:{controller-id}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrs>
    <ref>{ref-2}</ref>
    <snu>0</snu>
    <asid>{alert-session-id-2}</asid>
</mrs>
```

#### SIP Voice Channel Test Sequence

The voice channel test is initiated by the LUC but deliberately inverts the audio roles: the **ARC sends the test tone and the LUC echoes it back**, so the **ARC performs the quality assessment**. This keeps the LUC implementation minimal (embedded resources are limited) and ensures that signal evaluation is performed uniformly across devices from different manufacturers by a single authority.

On a successful run the ARC reports the test result to the LUC in the `Reason` header of the BYE. Whenever the LUC cannot complete the voice channel test successfully, it sends a regular `mrq` with `stc=0203` to the ARC — regardless of where the failure occurred. This covers in particular:

- the initial `MESSAGE` (1) is rejected (`snu ≠ 0`) or not answered at all,
- the `INVITE` (3) does not succeed (no `200 OK`, final 4xx–6xx response, or transaction timeout), so no media session is established,
- the ARC ends the session with a `BYE` (4) whose `Reason` indicates a failure (`cause=603`).

If the initial setup of the test fails — either the `MESSAGE` (1) or the `INVITE` (3) — the LUC should make a small number of additional attempts before reporting `stc=0203`, to avoid spurious failure reports caused by transient network conditions. A randomised wait interval between attempts is recommended so that simultaneous retries from many LUCs do not coincide. Once the BYE with `Reason=cause=603` has been received, the result is definitive and the LUC reports `stc=0203` without further retries.

This is intentional: the ARC is often a distributed system in which the media endpoint that runs the echo evaluation is not necessarily the component connected to the AMS. Routing the failure back through the LUC ensures that voice channel failures are reported on the same path as all other LUC-detected faults (alarm button, battery, internal bus, …).

```mermaid
sequenceDiagram
    participant LUC
    participant ARC

    Note over LUC: Scheduled or event-based audio self-test

    LUC->>ARC: (1) MESSAGE Request - Voice channel test
    Note right of ARC: Alarm identification and<br> location data received

    ARC->>LUC: (2) MESSAGE Response - Voice channel test
    Note left of LUC: Acknowledgement received.<br>Proceed to voice setup.

    LUC->>ARC: (3) INVITE sip:{echo}@domain.tld
    ARC->>LUC: 200 OK

    Note over ARC, LUC: SRTP channel established

	ARC->>LUC: Sends tone signal (non-DTMF)
	Note over LUC: Respond with echo
    LUC->>ARC: Receives echoed audio
    Note over ARC: Validate audio response

    ARC->>LUC: (4) BYE (result in Reason)
    LUC->>ARC: 200 OK
    
    opt On any failure (MESSAGE rejected, INVITE failed, or BYE Reason=fail)
        LUC->>ARC: (5) MESSAGE mrq - Voice channel (RTP) failure
        ARC->>LUC: (6) MESSAGE mrs OK
    end

```

##### (1) MESSAGE Request - Voice channel test

```xml
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrq>
    <ref>{ref-1}</ref>
    <mty>ME</mty>
    <cid>{controller-id}</cid>
    <stc>0202</stc>
    <asid>{alert-session-id}</asid>
</mrq>
```

##### (2) MESSAGE Response - Voice channel test

```xml
MESSAGE sip:{controller-id}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrs>
	<ref>{ref-1}</ref>
    <snu>0</snu>
    <cre>62</cre>
    <tnu>sips:{echo}@domain.tld</tnu>
    <asid>{alert-session-id}</asid>
</mrs>
```

##### (3) INVITE Request — Echo Test

```
INVITE sip:{echo}@domain.tld;transport=tls SIP/2.0
From: <sip:{controller-id}@domain.tld>;did={device-id}
To: <sip:{echo}@domain.tld>
Contact: <sip:{controller-id}@{ip}:{port};transport=tls;asid={alert-session-id}>
...
```

##### (4) BYE Request — Echo Test Termination

```
BYE sip:{echo}@domain.tld;transport=tls;asid={alert-session-id} SIP/2.0
Via: SIP/2.0/TLS {ip}:{port};branch=z9hG4bK-bye222
From: <sip:{controller-id}@domain.tld>;did={device-id};tag=abc123
To: <sip:{echo}@domain.tld>;tag=arc987
Reason: SIP;cause=603;text="Test failed"
...
```

The ARC conveys the test outcome to the LUC in the `Reason` header of the BYE (as defined in RFC 3326). The `cause` value indicates pass (`200`) or fail (`603`); the `text` field carries a short diagnostic. The following values are examples for orientation:

| Category                     | cause | text |
| ---------------------------- | ----- | ---- |
| Success                      | 200   | `Test passed` |
| Audio path                   | 603   | `Test failed: no RTP received` |
| Audio path                   | 603   | `Test failed: no echo received` |
| Audio path                   | 603   | `Test failed: one-way audio (no TX)` |
| Audio path                   | 603   | `Test failed: one-way audio (no RX)` |
| Audio path                   | 603   | `Test failed: RTP timeout` |
| Signal quality               | 603   | `Test failed: signal level too low` |
| Signal quality               | 603   | `Test failed: signal level too high (clipping)` |
| Signal quality               | 603   | `Test failed: SNR below threshold` |
| Signal quality               | 603   | `Test failed: excessive noise` |
| Tone/frequency detection     | 603   | `Test failed: tone not detected` |
| Tone/frequency detection     | 603   | `Test failed: frequency mismatch` |
| Tone/frequency detection     | 603   | `Test failed: tone distortion` |
| Tone/frequency detection     | 603   | `Test failed: THD exceeded` |
| Timing                       | 603   | `Test failed: echo delay too high` |
| Timing                       | 603   | `Test failed: jitter exceeded` |
| Timing                       | 603   | `Test failed: tone duration mismatch` |
| Transport                    | 603   | `Test failed: packet loss exceeded` |
| Transport                    | 603   | `Test failed: RTP sequence errors` |
| Transport                    | 603   | `Test failed: codec mismatch` |
| Flow/logic                   | 603   | `Test failed: ARC did not start tone sequence` |
| Flow/logic                   | 603   | `Test failed: echo loop not activated` |
| Flow/logic                   | 603   | `Test failed: test timeout` |
| Generic fallback             | 603   | `Test failed` |

##### (5) MESSAGE mrq - Voice channel (RTP) failure

```
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrq>
    <ref>{ref-2}</ref>
    <mty>ME</mty>
    <cid>{controller-id}</cid>
    <dty>0045</dty>
    <stc>0203</stc>
    <asid>{alert-session-id}</asid>
</mrq>
```

##### (6) MESSAGE mrs OK

```
MESSAGE sip:{controller-id}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrs>
    <ref>{ref-2}</ref>
    <snu>0</snu>
    <asid>{alert-session-id}</asid>
</mrs>
```

---

## lacs+xml Use Cases

The following use cases use `Content-Type: application/lacs+xml` for the extended communication. They always occur within the context of an active alarm session that was initiated via `scaip+xml`.

### Text-based Alarm Communication

If the LUC has an integrated TBUI, it must inform the ARC via the `cap` property in the lift voice alarm which options the TBUI supports.

> [!IMPORTANT]
>
> If the ARC or the LUC support TBUI, they must have implemented at least the simplest variant with predefined texts (`cap` must contain `0011`), which also serves as a fallback level.

As soon as a lift voice alarm in the corresponding local unit is initiated, the user must be asked question 1 (and then optionally question 2) immediately, with the answer options Yes and No.

These automated questions and their answers may only be transferred to the ARC after they have been answered, including a reference to the template (`aid`). As soon as the ARC has confirmed receipt with 200 OK, the user will see the message "Help request is being processed..."

If the TBUI has been operated and a rescue is initiated, a 'Help is on the way' with `aid=0131` must be sent from the ARC to the LUC.

The ARC is only allowed to send free text to the LUC if the `cap` contains `0012`.

#### Language Support

The TBUI must be able to display the predefined text templates in all supported languages, and the user must be able to select the appropriate language themselves.

If the user selects a different language, this language setting must be communicated to the ARC immediately if the TBUI supports receiving free text (`cap` contains `0012`). This notification is sent with `aid=0109` and `txt` containing the ISO 639-1 language code

The text property for messages is always additionally transmitted in English for predefined text templates (in order to remain human-readable). For free text communication, the user's last selected language must always be used.

#### Dynamic Answers

If supported by the TBUI device (`cap` contains `0013`), possible answers from the ARC can also be provided together with the free text (`aid`=`0100`). This occurs within the `text` property in the following format:

```
<Question>?<Answer 1>/../<Answer n>
```

At least 2 answer options must be provided and there must not be more than 4 answer options.

Sample:

```
How many people are in the lift?One/Two/Three/Four or more
```

#### Additional Real-time text (RTT)

In addition to the TBUI types defined here, LUC and ARC can optionally support real-time text (RTT) according to RFC 4103 in a separate SRTP channel and additionally provide this to the user.

#### Sequence Diagram

```mermaid
sequenceDiagram
    participant TBUI
    participant LUC
    participant ARC

    Note over TBUI: display anything
    Note over LUC: User presses alarm button

    par voice alarm
    	LUC->>ARC: (1) MESSAGE Request - Lift voice alarm
        ARC->>LUC: (2) MESSAGE Response - Lift voice alarm

        LUC->>ARC: (3) INVITE - initial call
        ARC->>LUC: 200 OK
        Note over LUC, ARC: Two-way speech/multimedia channel established

        ARC->>LUC: (4) BYE - initial call
        LUC->>ARC: 200 OK

    and text based communication
		Note over TBUI: User selects Language
        Note over TBUI: "Do you need help?<br>yes / no"
		Note over TBUI: User presses "yes"
		opt if implemented / configured
            Note over TBUI: "Do you need an ambulance?<br>yes / no"
            Note over TBUI: User presses "no"
        end

		LUC->>ARC: (5) MESSAGE Text-based dialogue - set language
        ARC->>LUC: MESSAGE mrs OK
        LUC->>ARC: (6) MESSAGE Text-based dialogue - question 1
        ARC->>LUC: MESSAGE mrs OK
        LUC->>ARC: (7) MESSAGE Text-based dialogue - answer 1
        ARC->>LUC: MESSAGE mrs OK
        opt if implemented / configured
            LUC->>ARC: (8) MESSAGE Text-based dialogue - question 2
            ARC->>LUC: MESSAGE mrs OK
            LUC->>ARC: (9) MESSAGE Text-based dialogue - answer 2
        end
        ARC->>LUC: MESSAGE mrs OK
        Note over TBUI: "Help request is<br> being processed..."
        opt possible if tbui-type >= 2
        	ARC->>LUC: (10) MESSAGE text: "Ne vous inquiétez pas,<br> nous travaillons à la libération.<br> Je vous recontacterai tout à l'heure." (free text)
        	LUC->>ARC: MESSAGE mrs OK
        	Note over TBUI: "Ne vous inquiétez pas,<br> nous travaillons à la libération.<br> Je vous recontacterai tout à l'heure."
        	Note over TBUI: User enters "Merci"
 	   		LUC->>ARC: (11) MESSAGE text: "Merci" (free text)
 	   		ARC->>LUC: MESSAGE mrs OK
        end
		ARC->>LUC: (12) MESSAGE Text-based dialogue - info 1
		LUC->>ARC: MESSAGE mrs OK
		Note over TBUI: "Help is on the way"
		opt possible if tbui-type >= 2
        	ARC->>LUC: MESSAGE text: "Le concierge sera<br> à l'ascenseur vers 15h35 et vous<br> libérera de la cabine." (free text)
        	LUC->>ARC: MESSAGE mrs OK
        	Note over TBUI: "Le concierge sera<br> à l'ascenseur vers 15h35 et vous<br> libérera de la cabine."
        end
	end

    Note over ARC,LUC: Voice/Multimedia Communication (as required)

    Note over LUC: EOA procedure
    Note over TBUI: display anything

```

##### (1) MESSAGE Request - Lift voice alarm

```xml
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrq>
    <ref>{ref-1}</ref>
    <mty>ME</mty>
    <mreq>1</mreq>
    <cid>{controller-id}</cid>
    <dty>0101</dty>
    <did>{device-id}</did>
    <stc>0010</stc>
    <lco>053</lco>
    <asid>{alert-session-id}</asid>
    <cap>0011,0012,0013,0014</cap>
</mrq>

```

##### (2) MESSAGE Response - Lift voice alarm

```xml
MESSAGE sip:{controller-id}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrs>
	<ref>{ref-1}</ref>
    <snu>0</snu>
    <mre>1</mre>
    <cre>62</cre>
    <tnu>sips:{arc-operator}@domain.tld</tnu>
    <asid>{alert-session-id}</asid>
</mrs>

```

##### (3) INVITE - initial call

```
INVITE sip:{arc-operator}@domain.tld;transport=tls

From: <sip:{controller-id}@domain.tld>;did={device-id}
To: <sip:{arc-operator}@domain.tld>
Contact: <sip:{controller-id}@{ip}:{port};transport=tls;asid={alert-session-id}>
...
```

##### (4) BYE - initial call

```
BYE sip:{controller-id}@{ip}:{port};transport=tls;asid={alert-session-id}
...
```

##### (5) MESSAGE Text-based dialogue - set language

```xml
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/lacs+xml

<mrq>
    <ref>{ref-2}</ref>
    <mty>TD</mty>
    <aid>0109</aid>
    <cid>{controller-id}</cid>
    <did>{device-id}</did>
    <dco>92</dco>
    <asid>{alert-session-id}</asid>
    <txt>fr</txt>
</mrq>

```

##### (6) MESSAGE Text-based dialogue - question 1

```xml
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/lacs+xml

<mrq>
    <ref>{ref-3}</ref>
    <mty>TD</mty>
    <aid>0111</aid>
    <cid>{controller-id}</cid>
    <did>{device-id}</did>
    <dco>92</dco>
    <asid>{alert-session-id}</asid>
    <txt>Do you need help?</txt>
</mrq>

```

##### (7) MESSAGE Text-based dialogue - answer 1

```xml
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/lacs+xml

<mrq>
    <ref>{ref-4}</ref>
    <mty>TD</mty>
    <aid>0121</aid>
    <cid>{controller-id}</cid>
    <did>{device-id}</did>
    <dco>92</dco>
    <asid>{alert-session-id}</asid>
    <txt>Yes</txt>
</mrq>

```

##### (8) MESSAGE Text-based dialogue - question 2

```xml
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/lacs+xml

<mrq>
    <ref>{ref-5}</ref>
    <mty>TD</mty>
    <aid>0112</aid>
    <cid>{controller-id}</cid>
    <did>{device-id}</did>
    <dco>92</dco>
    <asid>{alert-session-id}</asid>
    <txt>Do you need an ambulance?</txt>
</mrq>

```

##### (9) MESSAGE Text-based dialogue - answer 2

```xml
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/lacs+xml

<mrq>
    <ref>{ref-6}</ref>
    <mty>TD</mty>
    <aid>0122</aid>
    <cid>{controller-id}</cid>
    <did>{device-id}</did>
    <dco>92</dco>
    <asid>{alert-session-id}</asid>
    <txt>No</txt>
</mrq>

```

##### (10) MESSAGE Text-based dialogue - free text

```xml
MESSAGE sip:{controller-id}@domain.tld;transport=tls
...
Content-Type: application/lacs+xml

<mrq>
    <ref>{ref-7}</ref>
    <mty>TD</mty>
    <aid>0100</aid>
    <cid>{controller-id}</cid>
    <did>{device-id}</did>
    <dco>92</dco>
    <asid>{alert-session-id}</asid>
    <txt>Ne vous inquiétez pas, nous travaillons à la libération. Je vous recontacterai tout à l'heure.</txt>
</mrq>

```

##### (11) MESSAGE Text-based dialogue - free text

```xml
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/lacs+xml

<mrq>
    <ref>{ref-8}</ref>
    <mty>TD</mty>
    <aid>0100</aid>
    <cid>{controller-id}</cid>
    <did>{device-id}</did>
    <dco>92</dco>
    <asid>{alert-session-id}</asid>
    <txt>Merci</txt>
</mrq>

```

##### (12) MESSAGE Text-based dialogue - info 1

```xml
MESSAGE sip:{controller-id}@domain.tld;transport=tls
...
Content-Type: application/lacs+xml

<mrq>
    <ref>{ref-9}</ref>
    <mty>TD</mty>
    <aid>0131</aid>
    <cid>{controller-id}</cid>
    <did>{device-id}</did>
    <dco>92</dco>
    <asid>{alert-session-id}</asid>
    <txt>Help is on the way</txt>
</mrq>

```

### Camera support

Variants of camera support:

| cap value | Description                                       | Implementation                                                                                     |
| --------- | ------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| 0021      | CAMERA with still image (single snapshot)         | JPEG image as multipart SIP response see below                                                     |
| 0022      | CAMERA with video stream LUC -> ARC               | SDP + INVITE + SRTP - one stream with muted audio                                                  |
| 0023      | CAMERA with bidirectional video stream with audio | SDP + INVITE + SRTP - two streams with audio - <br /> can be switched during session with REINVITE |

#### Single Snapshot

> [!IMPORTANT]
>
> **JPEG Size Limitation**
>
> The JPEG image included in the multipart response must not exceed 1 MiB (1 048 576 bytes) in its binary form.
> Due to Base64 expansion (4/3), the maximum size of the Base64-encoded JPEG payload is 1 398 102 bytes.
> The LUC must ensure that the Base64-encoded block does not exceed this limit.

```mermaid
sequenceDiagram
    participant CAMERA
    participant LUC
    participant ARC


    Note over LUC: User presses alarm button


    LUC->>ARC: (1) MESSAGE Request - Lift voice alarm
    ARC->>LUC: (2) MESSAGE Response - Lift voice alarm

    LUC->>ARC: (3) INVITE - initial call
    ARC->>LUC: 200 OK
    Note over LUC, ARC: Two-way speech/multimedia channel established

    ARC->>LUC: (4) MESSAGE Request - Get camera snapshot
    Note over CAMERA: Capture snapshot
    LUC->>ARC: (5) MESSAGE Response - Camera snapshot

    ARC->>LUC: (6) BYE - initial call
    LUC->>ARC: 200 OK

    Note over ARC,LUC: Voice/Multimedia Communication (as required)
    Note over LUC: EOA procedure

```

##### (1) MESSAGE Request - Lift voice alarm

```xml
MESSAGE sip:{arc-reception}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrq>
    <ref>{ref-1}</ref>
    <mty>ME</mty>
    <mreq>1</mreq>
    <cid>{controller-id}</cid>
    <dty>0101</dty>
    <did>{device-id}</did>
    <stc>0010</stc>
    <lco>053</lco>
    <asid>{alert-session-id}</asid>
    <cap>0021</cap>
</mrq>

```

##### (2) MESSAGE Response - Lift voice alarm

```xml
MESSAGE sip:{controller-id}@domain.tld;transport=tls
...
Content-Type: application/scaip+xml

<mrs>
	<ref>{ref-1}</ref>
    <snu>0</snu>
    <mre>1</mre>
    <cre>62</cre>
    <tnu>sips:{arc-operator}@domain.tld</tnu>
    <asid>{alert-session-id}</asid>
</mrs>

```

##### (3) INVITE - initial call

```
INVITE sip:{arc-operator}@domain.tld;transport=tls

From: <sip:{controller-id}@domain.tld>;did={device-id}
To: <sip:{arc-operator}@domain.tld>
Contact: <sip:{controller-id}@{ip}:{port};transport=tls;asid={alert-session-id}>
...
```

##### (4) MESSAGE Request - Get camera snapshot

```xml
MESSAGE sip:{controller-id}@domain.tld; transport=tls; did={device-id}

...
Content-Type: application/lacs+xml

<mrq>
    <ref>{ref}</ref>
    <mty>CO</mty>
    <aid>0200</aid>
    <cid>{controller-id}</cid>
    <did>{device-id}</did>
    <asid>{alert-session-id}</asid>
</mrq>
```

##### (5) MESSAGE Response - Camera snapshot

```xml
MESSAGE sip:{arc-reception}@domain.tld; transport=tls

...
Content-Type: multipart/mixed; boundary="lacs-boundary-001"
Content-Length: 950000

--lacs-boundary-001
Content-Type: application/lacs+xml; charset=utf-8
Content-Transfer-Encoding: 8bit

<mrs>
	<ref>{ref}</ref>
	<snu>0</snu>
</mrs>

--lacs-boundary-001
Content-Type: image/jpeg
Content-Transfer-Encoding: base64

{base64-encoded-JPEG}

--lacs-boundary-001--
```

##### (6) BYE - initial call

```
BYE sip:{controller-id}@{ip}:{port};transport=tls;asid={alert-session-id}
...
```
