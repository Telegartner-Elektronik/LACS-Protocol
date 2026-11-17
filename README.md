# LACS-Protocol

Lift Alarm Communication Standard: lift alarm communication according to EN 81-28 with SIP signalling and voice over RTP, based on CLC/TS 50134-9 (SCAIP).

You can find the current release candidate of the specification in [`LACS.md`](./LACS.md). It consists of the following documents:

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

The XML schemas use XML Schema 1.1 (assertions). Validate with an XML Schema 1.1 processor, e.g. Saxon EE, Xerces-J or Python `xmlschema`; libxml2 (`xmllint`), lxml and .NET support XML Schema 1.0 only.
