# Changelog

All notable changes to the AURIORA Software Style Guide are documented in this file. Released versions are tagged in version control.

## 0.2.0 - 2026-09-12

- Module fleet rules (§3.5): key the device registry on the Module's persistent identity as reported over the Module Control Interface, never on a device node, port path or hub position, and hold physical location as a mutable attribute of the identified Module so that moving one updates its location while replacing one is recognized as a different device; take feature decisions from reported capabilities rather than a product-number table; query the Module's inventory and content integrity values and transfer only what differs; express experiment setups as desired state and diff against what each Module reports; keep logical groups host-side, where they work identically for locally connected and hub-connected Modules; confirm every required Module actually armed and refuse to proceed by name when one did not, and never use a control command as the simultaneous trigger for several Modules — the deterministic instant is a SYNC event; treat discovery as establishing identity and nothing more; record every participating Module's identity, firmware version and configuration plus the SYNC topology in the experiment record; and let the simulated device of §3.4 speak the same control semantics. The interface itself is defined by AES 0.7.0 (`AES-MCI-001` to `AES-MCI-005`).

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
