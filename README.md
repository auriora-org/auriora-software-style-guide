# AURIORA Software Style Guide

The AURIORA Software Style Guide (ASSG) is the official software development guide for all AURIORA host-side software: desktop and server applications, command-line tools, test and manufacturing tooling, services and user interfaces. It complements the AURIORA Engineering Standard (AES): AES defines *what* must be true about AURIORA engineering work, ASSG defines *how* software should be designed, structured, implemented and maintained.

Requirements scale with the AES maturity level: damage-prevention rules apply always (validated untrusted input near devices and data, no secrets in repositories, integrity-checked artifacts before they touch hardware), release-grade obligations (reproducible builds, packaging, declared platform support) bind Released software, and the layered architecture is the model tools converge on as they mature — **a small script is allowed to remain a small script**.

Read the guide: [GUIDE.md](./GUIDE.md)

## Scope

- Design philosophy and a common software architecture: layers, modules, dependency injection, pure logic separation
- Device communication: the protocol specification as contract, layering, robustness, simulated devices
- Project structure, languages and toolchains, coding style, API design, error handling and concurrency
- Dependency management, configuration, logging and diagnostics, CLI/UI conventions, data formats
- Security, testing and packaging scaled to maturity, versioning and compatibility, documentation
- A compact code review checklist with maturity-scaled depth

The guide is language-independent: it defines the architecture, discipline and quality bar that apply in every programming language, and delegates syntax-level style to each language's canonical conventions. Embedded firmware is covered by the AURIORA Firmware Style Guide.

## Requirement Language

Aligned with AES:

- `MUST` — mandatory when applicable (equivalent to `SHALL` in AES)
- `SHOULD` — recommended default; engineering judgment may justify another approach, no documented exception needed
- `MAY` — optional improvement

Where ASSG and AES conflict, AES prevails.

## Versioning and Changes

Released versions are recorded in the [CHANGELOG](./CHANGELOG.md) and tagged in version control.

## License

This documentation is licensed under the [Creative Commons Attribution-ShareAlike 4.0 International License (CC BY-SA 4.0)](./LICENSE).
