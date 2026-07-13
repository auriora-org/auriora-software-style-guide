# Changelog

All notable changes to the AURIORA Software Style Guide are documented in this file. Released versions are tagged in version control.

## 0.1.0 - 2026-07-13

First release.

- Design philosophy and a maturity-scaling model: damage-prevention rules always; reproducible builds, packaging and platform-support claims bind Released software; the layered architecture is the model tools converge on as they mature. A small script is allowed to remain a small script.
- Software architecture: layered model (presentation, application, device communication, transport), module responsibilities, dependency injection, pure logic separation.
- Device communication: protocol specification as the contract, layering and isolation, robustness rules, simulated devices for development and testing.
- Project structure, language and toolchain policy, coding style, API design and error handling guidance.
- Concurrency, dependency management and configuration rules.
- Logging and diagnostics including wire-level protocol tracing (required for Released device tooling).
- Command-line and user interface conventions, data format and storage rules.
- Security, testing, packaging and distribution requirements scaled to maturity.
- Versioning and device compatibility rules aligned with the AURIORA Engineering Standard.
- Documentation requirements aligned with the AURIORA Documentation Standard.
- A compact code review checklist (every change / device communication / toward Release); self-review explicitly valid.
- Requirement language aligned with AES (MUST/SHOULD/MAY; a skipped SHOULD needs no documented exception).
- Language-independent policy: no language-, framework- or vendor-specific rules.
- CC BY-SA 4.0 license.
