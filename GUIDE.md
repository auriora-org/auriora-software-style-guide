# AURIORA Software Style Guide

**Document ID:** ASSG
**Version:** 0.1.0
**Status:** Normative
**Complements:** AURIORA Engineering Standard (AES)
**Language:** English

## About This Guide

This is the official software development guide for all AURIORA host-side software: desktop and server applications, command-line tools, test and manufacturing tooling, services and user interfaces — everything that runs on a general-purpose computer rather than on the embedded target. It complements the AURIORA Engineering Standard: AES defines *what* must be true about AURIORA engineering work, while this guide defines *how* software should be designed, structured, implemented and maintained. Embedded firmware is covered by the AURIORA Firmware Style Guide; documentation by the AURIORA Documentation Standard.

AURIORA software is written in different languages for different purposes. This guide is therefore language-independent: it defines the architecture, discipline and quality bar that apply in every language, and delegates syntax-level style to each language's canonical conventions (Section 5). Principles age well; language fashions do not.

Requirement language (aligned with AES):

- **MUST** — mandatory when applicable. Equivalent to `SHALL` in AES.
- **SHOULD** — recommended default; engineering judgment may justify another approach. Skipping a SHOULD requires no documented exception.
- **MAY** — optional improvement.

Where this guide and AES conflict, AES prevails. Deviating from an applicable MUST needs a concise note in the design notes; formal exception records are reserved for Released artifacts and platform-wide deviations ([AES-GOV-003](https://github.com/auriora-org/auriora-engineering-standard/blob/main/STANDARD.md#aes-gov-003-honest-deviations)).

### Maturity Scaling

Requirements scale with the [AES maturity level](https://github.com/auriora-org/auriora-engineering-standard/blob/main/STANDARD.md#3-maturity-model):

- **Always (all maturity levels):** honesty and damage prevention — validated untrusted input where it can reach a device or persistent data, no secrets in repositories, integrity-checked artifacts before they touch hardware, honest exit codes for anything scripts already depend on.
- **Released only:** release-grade obligations — reproducible builds from a tagged commit, packaging and installation paths, declared and tested platform support, versioned public contracts, complete user-facing documentation.
- **Everything else** is the recommended default (SHOULD). **A small script is allowed to remain a small script**: a one-file internal tool needs no layered architecture, no `docs/` directory, no CI, no changelog and no formal decision records unless it grows or becomes externally maintained. The architecture below is where tools converge as they mature.

---

## 1. Design Philosophy

AURIORA software exists to serve the hardware platform for its whole lifetime — it will be maintained, ported and extended long after the laptops it was written on are recycled.

- **Simplicity.** The simplest design that meets requirements is the correct design. Frameworks, layers and abstractions MUST be justified by a real requirement. A script that does the job beats a platform that might.
- **Readability.** Code is read far more often than written, frequently by someone fluent in a different language than the author's favourite. Prefer plain constructs over language tricks; clever code is a tax on every future reader.
- **Reliability.** Software that talks to hardware fails in hardware ways: unplugged cables, half-written responses, devices that reboot mid-transaction. Every external interaction is designed for failure first (Section 3).
- **Maintainability.** A module SHOULD be understandable from its public interface and documentation alone. Non-obvious decisions are documented where they apply; decisions that are expensive to reverse are worth an ADR per the AURIORA Documentation Standard — routine ones are not.
- **Explicitness over cleverness.** Dependencies, state and side effects are visible: explicit injection, explicit configuration, explicit errors. Magic — hidden globals, implicit conversions, action-at-a-distance metaprogramming — MUST be avoided.
- **Predictability.** Same input, same state, same behaviour — across runs, machines and operating systems. Uniform structure across AURIORA repositories means a developer who knows one tool can navigate all of them.
- **Long-term support.** Languages and libraries churn faster than hardware platforms live. Keep the dependency surface small (Section 10) and the device knowledge in project-owned code, so the software survives its ecosystem's fashions.
- **Open source philosophy.** Software MUST be buildable and runnable from its published repository with documented, preferably free toolchains, on the supported platforms. Write for strangers — they are the future maintainers.

---

## 2. Software Architecture

The same architectural discipline that governs AURIORA firmware governs its software: layers with one-way dependencies, modules with single responsibilities, and platform knowledge kept out of the application logic.

Scaling: this architecture binds software of real complexity — anything with a user interface, multiple workflows or a maintained lifespan — and Released device tooling. A small script does the job in one file and stays that way until growth proves otherwise.

- **Layered model.** Software that interacts with AURIORA hardware SHOULD separate at least these concerns, with dependencies pointing downward only:

| Layer | Responsibility |
|---|---|
| **Presentation** | CLI, GUI or API surface — parsing input, rendering output, nothing else |
| **Application** | Workflows and use cases: what the tool actually does |
| **Device communication** | Protocol implementation and device/session management (Section 3) |
| **Transport** | Serial, USB, network, BLE — opening, closing, moving bytes |

- **Presentation is replaceable.** Business and device logic SHOULD NOT live in UI handlers or CLI argument code. The test: the tool's core could be driven by a different frontend (CLI today, GUI or CI harness tomorrow) without touching the logic.
- **One module, one responsibility.** Modules expose a deliberate public interface and hide everything else. If a module's purpose cannot be stated in one sentence, split it.
- **Dependency injection.** Modules receive their collaborators (transports, devices, storage, clocks) at construction — not by reaching for globals or constructing them internally. This is what makes the testing strategy of Section 16 possible.
- **Pure logic stays pure.** Parsing, validation, computation and decision logic SHOULD be separated from I/O so it can be tested without devices, files or network. This mirrors the firmware guide's policy/mechanism split — the payoff is identical.
- **Architecture documentation.** Non-trivial repositories in Active Development or beyond SHOULD document their architecture — components, responsibilities, data flow, structural decisions — and Released tools MUST. A section in `docs/design-notes.md` is a fine home.

---

## 3. Device Communication

Talking to hardware is what distinguishes AURIORA software from generic software — and it is where generic software habits fail. This chapter is the counterpart of the firmware guide's interface rules: the two ends of the cable follow one discipline.

### 3.1 The Protocol Is the Contract

- **One canonical specification.** Every protocol between software and an AURIORA device MUST have a single canonical interface specification (per the Documentation Standard), versioned and referenced by both the firmware and the software implementation. Neither implementation is the reference — the document is. When behaviour and specification disagree, that is a bug in one of them, resolved explicitly.
- **Protocol code is generated or centralized.** Message definitions (IDs, layouts, units, error codes) MUST exist in exactly one place in the codebase — a protocol module or generated code — never hand-copied constants scattered through application logic. Duplicated protocol knowledge diverges silently.
- **Version negotiation.** The software MUST discover the device's protocol/firmware version before relying on version-specific behaviour, and MUST handle unsupported versions deliberately: refuse clearly, or degrade to a documented subset — never guess. Compatibility claims name exact versions.

### 3.2 Layering and Isolation

- **Device communication is a library, not an application.** Protocol implementation, device discovery and session management live in a dedicated module (or package) with a clean API, independent of any UI. Application code says `device.read_calibration()` — it MUST NOT build frames, know message IDs or touch the transport.
- **Transport-agnostic protocol.** The protocol layer speaks to an abstract transport interface (open/close/read/write with timeouts), so serial, USB, TCP or a simulated device (Section 3.4) are interchangeable underneath it.
- **Device state lives in one place.** A session object owns the connection, its state and its invariants. Passing raw handles around the codebase — every holder a potential corruptor — is a design defect.

### 3.3 Robustness

- **Timeouts everywhere.** Every device operation MUST have a timeout with a defined failure result. No call may block forever because a cable was pulled — the user's next action must always be possible.
- **Expect disconnection at any byte.** Devices unplug, reboot and brown-out mid-transaction. The communication layer MUST recover to a defined state from partial reads, garbage bytes and lost connections: resynchronize, report, and support deliberate reconnection. State on both ends after a failed transaction MUST be defined, not assumed.
- **Validate everything received.** Framing, length, CRC, ranges — the same paranoia the firmware applies to input (its Section 14) applies in reverse. Malformed device data is counted, reported and discarded; it MUST NOT crash the tool or corrupt stored results.
- **Retries are deliberate.** Retry only idempotent operations, with bounded attempts and backoff, and log every retry — a link that needs retries is a diagnosis waiting to be made. Commands with side effects (erase, calibrate, update) are never blindly retried.
- **The user sees the truth.** Device errors surface as meaningful messages (what failed, on which device, likely cause), not stack traces or silent hangs. Long operations report progress and remain cancellable.

### 3.4 Development Without Hardware

- **A simulated device SHOULD exist** for any protocol of real complexity, and for Released tooling whose protocol layer needs hardware-free testing it is the expected mechanism: a software implementation of the device side, speaking the same protocol over the same transport interface. Tests run against it (Section 16); developers work without bench hardware; fault cases impossible to produce on demand with real devices (corruption, timeouts, version mismatches) become ordinary test cases.
- **The simulator follows the specification,** not the firmware's quirks — that is how specification drift gets caught from the second direction.
- **Record/replay MAY be used** to capture real device sessions for regression tests where full simulation is disproportionate.

---

## 4. Project Structure

Layout varies by language ecosystem — fighting the ecosystem's conventions costs more than it buys. What SHOULD be recognizable as a repository grows:

- **Standard entry points.** `README.md` and `LICENSE` always; `CHANGELOG.md`, `docs/` and the rest per the AURIORA Documentation Standard — created when their content exists. A newcomer reaches a working build from the README alone.
- **Source in one root** (`src/` or the ecosystem's equivalent), organized by the layers of Section 2 where those layers exist, so the architecture is visible in the directory tree. A single-file script is its own root.
- **Tests mirror sources** in the ecosystem's standard test location, runnable with one documented command.
- **Tooling committed.** Formatter, linter and build configuration files live in the repository where they are used (Section 5); developer scripts in `tools/`.
- **No generated artifacts in version control** (MUST). Build outputs, caches and virtual environments stay out; generated protocol code is either committed with its generator and inputs, or generated by the build — one or the other, stated in the README.

---

## 5. Languages and Toolchains

AURIORA does not maintain per-language style guides. It maintains one rule and enforces it mechanically.

- **Choose boring, durable languages.** A new project SHOULD use a language already established in the AURIORA ecosystem; adding a language to the ecosystem is a permanent maintenance commitment (toolchain, tooling, reviewer competence) worth a short design-notes entry stating why — no formal ADR required for a reasonable choice.
- **Follow the language's canonical style.** Code SHOULD follow the official or de-facto canonical style conventions and formatting tools of its language, whatever that language is. AURIORA does not override language communities on syntax — consistency *within* the ecosystem beats consistency *across* languages.
- **Automate it.** Repositories in Active Development or beyond SHOULD commit their formatter and linter configuration and apply them mechanically (in CI where it exists). Style is enforced by tools, reviewed by nobody: humans review design, not indentation.
- **Pin the toolchain.** For anything beyond Experimental, required language version, build tool versions and setup steps are declared in the repository. "Works with whatever is installed" is not reproducible (Section 10 governs dependencies).
- **Static analysis on.** Compiler warnings, type checkers and standard linters SHOULD run at a strict, recorded setting; treat warnings as errors. In gradually-typed languages, public interfaces SHOULD be fully typed.

---

## 6. Coding Style

Language-independent rules — the ones that survive translation.

- **Names carry meaning.** English, specific, pronounceable; functions read as verb phrases, booleans as assertions, and units live in names or types (`timeout_ms`, `Millivolts`). The same concept has the same name across the whole codebase — and matches the protocol specification's terms where one applies.
- **No magic numbers.** Every non-obvious literal is a named constant; protocol values come from the protocol module (Section 3.1), never inlined.
- **Small units.** Short functions with one job; modules that fit in the head. Deep nesting flattens into early returns; long parameter lists become configuration objects.
- **Immutability by default.** Prefer immutable data and pure functions; confine mutation to clearly-owned state. Shared mutable state is the same bug source on a workstation as on a microcontroller.
- **No dead code.** Commented-out blocks, unreachable branches and unused flags MUST NOT be committed — version control remembers.
- **Comments explain why.** The workaround's reason, the protocol erratum, the non-obvious constraint — with references. What the code does is the code's job to say.

---

## 7. API Design

These rules apply to every public interface: library APIs, internal module boundaries, and service endpoints alike.

- **Minimal, deliberate surface.** Export only what callers need; everything else is private by the language's strongest available mechanism. Growing an API is easy, shrinking one is a breaking change.
- **Consistent shape.** Construction takes configuration; operations return results or raise defined errors; lifecycle (open/close, context management) follows the ecosystem's idiom. The same concept has the same name and shape in every AURIORA API.
- **Honest signatures.** Inputs and outputs are explicit — no hidden reliance on globals, environment or ambient state. In typed languages, types tell the truth; in untyped ones, documentation does (per the Documentation Standard's code rules).
- **Errors are part of the API.** Every failure a caller can meaningfully handle is a defined, documented error type — not a string to parse (Section 8).
- **Compatibility is versioned.** Public APIs consumed outside the repository follow the AES versioning standard: breaking changes bump the major version and are called out in the changelog. Within a repository, changing an interface means updating all callers in the same change.

---

## 8. Error Handling

Tools that drive hardware are used at the worst moments — bring-up, manufacturing, field debugging. Their error handling is the user experience.

- **Use the language's idiomatic mechanism** (exceptions, result types, error values) — consistently, one strategy per codebase. Do not fight the language; do not mix paradigms.
- **No swallowed errors.** Every failure is handled, propagated or explicitly and visibly ignored with a documented reason. Empty catch blocks and discarded results are review defects.
- **Fail with context.** An error that reaches the user states what was attempted, what failed and — where known — what to do about it: `"Failed to open /dev/ttyACM0: permission denied (is your user in the dialout group?)"`, not `Errno 13`.
- **Errors carry structure.** Device errors, usage errors and internal bugs are distinguishable types, because they demand different responses: retry, fix the invocation, file a report.
- **Validate at the boundaries:** user input, files, network data and device responses (Section 3.3) are checked on entry. Inside the boundary, trust your own invariants and assert them — a failed assertion is a bug, reported as one.
- **Exit honestly.** Processes exit non-zero on failure, zero only on success — scripts and CI depend on it. Partial success is failure unless the interface explicitly defines otherwise.
- **Clean up deterministically.** Connections, files, locks and temporary artifacts are released on every path, including failure paths — use the language's scoped-resource idiom. A crashed tool MUST NOT leave a device in a state that requires power-cycling to recover, where the protocol permits better.

---

## 9. Concurrency

- **Concurrency is a design decision,** not a sprinkle. Introduce it for a stated reason — responsiveness, throughput, multiple devices — and contain it behind module boundaries; most code stays sequential and oblivious.
- **Prefer message passing and task models** (queues, async/await, worker pools) over shared mutable state with locks — the same single-ownership rule as the firmware guide: every piece of mutable data has one owner.
- **The UI never blocks.** Interactive tools keep device I/O off the interaction thread; long operations report progress and support cancellation (Section 3.3).
- **One session, one owner.** Concurrent access to a device session is serialized by its owner; two threads interleaving writes on one transport is a protocol corruption generator.
- **Cancellation is designed,** not improvised: long operations define where they can stop and what state they leave behind.
- **Document the contract.** Every public API states whether it is thread-safe/task-safe. Absence of a statement means it is not.

---

## 10. Dependency Management

Every dependency is code you now maintain without controlling. AURIORA software is expected to build in ten years; its dependency tree is chosen accordingly.

- **Minimal by policy.** A dependency MUST earn its place: substantial functionality, maintained, suitable license. Trivial conveniences are written, not imported. Prefer the language's standard library wherever it is adequate.
- **License compliance.** Every dependency's license MUST be compatible with the project's license and AURIORA's open-source publication. Licenses are checked on adoption, not discovered at release.
- **Pin and lock.** Applications commit lockfiles; builds are reproducible from the repository alone. Version updates are deliberate changes — reviewed, tested, changelogged — not side effects of rebuilding.
- **Isolate the risky ones.** Dependencies likely to churn or die (vendor SDKs, GUI frameworks, niche libraries) sit behind project-owned interfaces, so replacement is a module change, not a rewrite. The transport abstraction of Section 3.2 is this rule applied to hardware access.
- **Update deliberately.** Dependencies are reviewed on a cadence and updated in small, tested steps; security advisories for pinned versions SHOULD be monitored mechanically. The riskiest dependency is the one nobody has touched in five years and nobody dares to.

---

## 11. Configuration

- **Layered precedence, documented:** built-in defaults ← configuration file ← environment ← command-line flags. Later layers override earlier; the effective configuration MUST be inspectable (a `--show-config` equivalent SHOULD exist).
- **Sane defaults.** The common case runs with no configuration at all. Every option has a documented default, valid range and effect (per the Documentation Standard); an option nobody documented is an option nobody dares change.
- **Validate on load.** Configuration is checked completely at startup — unknown keys, bad values and conflicts are reported with the file and field named, before any device is touched. Failing at step one beats failing at step forty.
- **Configuration files are versioned artifacts:** text-based, diffable, with a schema or reference documentation, and migrated or safely defaulted across versions.
- **No secrets in configuration files** that enter version control — keys and credentials arrive by environment or platform keychain (Section 15).

---

## 12. Logging and Diagnostics

The bug report you will get is "it doesn't work". Logging is how the tool testifies about what actually happened.

- **Standard levels, used consistently:** `ERROR` (operation failed), `WARNING` (degraded, recovered), `INFO` (what the tool is doing, at human pace), `DEBUG` (detail for diagnosis, including protocol traffic). Verbosity is user-selectable (`-v`, `--log-level`); default output is quiet enough to respect the user and complete enough to reconstruct a failure.
- **Output discipline.** Results go to stdout, diagnostics to stderr — pipelines depend on the separation (Section 13). Interactive progress display is presentation, not logging; the two are separable.
- **Wire-level tracing SHOULD be available** for every device protocol the tool speaks — raw frames, decoded meaning, timestamps, direction, switchable at runtime — and Released device tooling MUST provide it. Protocol bugs between two implementations are undebuggable without seeing the actual bytes; this single feature repays its cost at the first field incident.
- **Structured where it matters.** Logs that machines will consume (services, CI, fleet tooling) SHOULD be structured (e.g. JSON lines); logs for humans stay human-readable. Timestamps in ISO 8601, UTC or offset-explicit.
- **Diagnostics are a feature.** Tools SHOULD expose their own health facts: version (Section 18), connected device identity and firmware version, session statistics, error counters. The support question "what does `tool info` say?" beats twenty speculative ones.
- **Never log secrets.** Credentials, keys and personal data MUST NOT reach logs at any level (Section 15).

---

## 13. Command-Line and User Interfaces

Most AURIORA software meets its users as a CLI; some as a GUI. Both are contracts.

- **CLI conventions.** Follow platform norms: `tool command [options] [args]` for multi-function tools; long options for every flag (short forms for the frequent ones); `--help` that is accurate, `--version` that is honest. Flags, not positional guesswork, for anything non-obvious.
- **Exit codes are API** (Section 8): zero on success, documented non-zero on failure. Scripts and manufacturing fixtures will depend on them.
- **Machine-readable output on request.** Any command whose output other tools will consume SHOULD offer a stable structured format (`--json` or equivalent). Human-readable output MAY change freely; structured output is versioned API.
- **Non-interactive by default.** Every operation MUST be scriptable: no mandatory prompts, no "press any key". Interactive confirmation is reserved for destructive operations and MUST be bypassable (`--yes`) for automation.
- **Destructive operations are explicit:** named unambiguously (`erase`, not `clean`), confirmed interactively, and stated clearly in `--help`. The flag that erases calibration data is never one typo away from the flag that reads it.
- **GUIs follow the same separation:** the interface drives the same application layer as the CLI (Section 2), long operations show progress and allow cancellation, and errors surface with the same context and honesty as Section 8 demands.

---

## 14. Data Formats and Storage

Software that measures, calibrates and tests produces data whose value outlives the tool that wrote it.

- **Open, documented formats.** Data intended to persist MUST use open, text-based-where-practical formats with a documented schema — readable in ten years without this tool. Proprietary or undocumented binary formats are a data loss with a delay.
- **Version every schema.** Files carry a format version; readers accept what they can, reject what they can't — explicitly, never by misinterpreting. Migrations are written when formats change, not when users complain.
- **Provenance travels with data.** Measurement and calibration records carry their context: tool version, device identity and firmware version, timestamp, relevant configuration. An unattributed measurement is an anecdote.
- **Write atomically.** Files are written completely-or-not-at-all (write-rename, transactions); a crash mid-write MUST NOT corrupt existing data. Data that is expensive to reproduce (calibration!) deserves paranoia proportional to its cost.
- **Units are explicit** in schemas and field names, per the same rule the protocol specifications follow. A column named `voltage` is a question; `voltage_mv` is an answer.

---

## 15. Security

Host software's threat model differs from firmware's: it handles developer machines, networks, credentials and update artifacts. Proportionality still rules — but the threat model MUST be written down.

- **No secrets in repositories** — not in code, configuration, test fixtures or CI logs. Credentials arrive by environment, platform keychains or dedicated secret stores; a leaked-secret incident includes rotation, not just deletion.
- **Validate untrusted input** with the same discipline applied to device data (Section 3.3): files, network responses and user input are checked before use. Anything that parses complex input from outside is a security surface.
- **Dependencies are attack surface** (Section 10): sources are the official registries, versions are pinned, advisories monitored. Supply-chain hygiene is part of dependency policy, not an afterthought.
- **Artifacts that touch devices are verified.** Firmware images and update packages handled by tooling MUST be integrity-checked (and signature-verified where the platform defines signing) before being sent to a device — the tooling is the last check before hardware.
- **Network communication** beyond localhost uses authenticated, encrypted standard protocols where the threat model includes the transport. Homemade cryptography is prohibited, same as in firmware.
- **Least privilege.** Tools request the access they need (device permissions, file scopes) and no more; nothing runs as root/administrator that does not have to, and anything that must says why in its documentation.

---

## 16. Testing

The architecture of Sections 2–3 exists to make software testable without a lab. Use that — scaled to maturity and risk: an experimental script needs no test suite; a Released calibration tool earns its trust with one.

- **Unit tests for the pure core.** Parsing, computation, validation and protocol encoding/decoding SHOULD be tested as pure functions — normal cases, boundaries, malformed input — and for Released tooling the critical paths MUST be. This is where most bugs die, in milliseconds.
- **Protocol tests against the simulator.** Where a simulator exists (Section 3.4), the device-communication layer is tested against it: full sessions, version mismatches, timeouts, disconnections mid-transaction, corrupted frames. Fault cases that hardware cannot produce on demand are the simulator's whole point.
- **Integration tests with real hardware** verify what simulation cannot: timing, transport quirks, actual firmware behaviour. They run before releases and on hardware-affecting changes; where volume justifies it, automated on a hardware-in-the-loop rig.
- **Regression tests for every fixed bug** worth not fixing twice — a bug fixed without a test will return.
- **Test across supported platforms.** For Released software, the support claim in the README (Section 17) is a test matrix, not an aspiration: cover the supported operating systems and language versions.
- **CI** SHOULD gate merges for Active Development and beyond — format, lint, static analysis, unit and simulator tests, warning-free — and one documented command runs the whole suite locally. Experimental repositories need no CI; the local command carries the obligation.

---

## 17. Packaging and Distribution

This section binds **Released** software — internal tools and experiments are installed by the people who wrote them. Software nobody can install is software nobody uses — and "clone the repo and fight the toolchain" is not installation.

- **One blessed installation path** per audience, documented in the README: the ecosystem's standard package format for end users, a documented development setup for contributors. Both reach a working state from a clean machine.
- **Supported platforms are named.** The README states exactly which operating systems and versions are supported; each is covered by the test matrix (Section 16). Everything else is explicitly best-effort.
- **Releases are reproducible:** built from a tagged commit with pinned dependencies — by CI where it exists, otherwise by documented steps with recorded artifact checksums. Where the ecosystem supports signing, artifacts are signed.
- **Runtime dependencies install with the package.** The user installs the tool, not the tool plus a scavenger hunt. System-level prerequisites that cannot be bundled (drivers, device access permissions) are documented and, where possible, automated.
- **Uninstallation is clean:** the package removes what it installed, and documentation says what user data (configuration, measurements) remains and where.

---

## 18. Versioning and Compatibility

- **Versions follow the AES versioning standard** (semantic versioning): breaking changes to any public contract — CLI, API, file formats, structured output — bump the major version. The changelog (Keep a Changelog, per the Documentation Standard) records every release.
- **The tool knows its own version.** `--version` reports the exact released version plus build traceability (VCS revision); the same identity appears in logs (Section 12) and in data provenance (Section 14). An artifact of unknown origin is undebuggable.
- **Device compatibility is declared.** Each software release states which device firmware/protocol versions it supports (Section 3.1), in the README and in release notes. The support window is a policy — decided, documented and tested — not an accident of what happened to still work.
- **Deprecate before removing:** deprecated commands, options and APIs warn at use, are marked in documentation, and survive at least one minor release before removal. Users read changelogs after things break; the tool itself is the effective notice.

---

## 19. Documentation

Software documentation follows the AURIORA Documentation Standard; minimums are set by the AES maturity level. The software-specific shape:

- **README:** overview, maturity status, usage; for Released software also supported platforms, installation and a first working example. For a small internal tool the README is the entire documentation, and that is correct.
- **User documentation is task-oriented** (Released tools with real users): the workflows users actually perform — flash a device, run a calibration, export data — each with the command, the expected result and the likely failures. Organized by goal, not by module.
- **Every command, option and exit code of Released tools documented** — `--help` and the reference documentation agree with each other and with the implementation.
- **API documentation** per the Documentation Standard for anything consumed as a library: contracts, errors, thread-safety, runnable examples.
- **Interface specifications** for every protocol and persistent format the software speaks (Sections 3, 14) — citable by Release; a current table in the design notes before that.
- **Architecture and decision notes** per Section 2 — language, framework and dependency choices are worth a line in the design notes (or an ADR when expensive to reverse), because they are exactly the decisions future maintainers question first.

---

## 20. Code Review Checklist

Compact checklist for software changes. Self-review is valid ([AES-QA-001](https://github.com/auriora-org/auriora-engineering-standard/blob/main/docs/07-maturity-and-release.md#aes-qa-001-proportionate-review)) — preferably after a short time away from the code. Skip items that plainly don't apply; no N/A bookkeeping. The first block applies to every change; the rest bears down as maturity rises.

**Every change**
- [ ] Clean build/run from documented steps
- [ ] Critical error paths considered: failures handled or visibly propagated, no empty catches, exit codes honest
- [ ] Untrusted input (user, file, network, device) validated where it can reach a device or persistent data
- [ ] Interfaces match their documentation (`--help`, formats, protocol tables)
- [ ] Obvious secrets and generated artifacts excluded
- [ ] Known limitations and non-obvious decisions recorded (design notes or comments)
- [ ] Relevant basic tests pass

**Device communication (where touched)**
- [ ] Protocol knowledge only in the protocol module; matches the specification
- [ ] Every device operation has a timeout and defined failure behaviour
- [ ] Disconnection, partial data and malformed responses handled to a defined state
- [ ] Resources (ports, sessions, files) released on all paths, including failure and cancellation
- [ ] Simulator updated alongside protocol changes, where one exists

**Toward Release (additionally)**
- [ ] Formatter, linter, static analysis and type checks pass; no new warnings
- [ ] Concurrency additions justified; ownership and thread-safety contracts stated
- [ ] New dependencies justified, license-checked, pinned; lockfile updated deliberately
- [ ] CLI/API/format changes versioned correctly; breaking changes flagged and changelogged
- [ ] Destructive operations explicit and confirmed; automation bypass exists
- [ ] New logic has unit tests; fixed bugs have regression tests; supported-platform claims still hold
- [ ] README, user docs and changelog updated for user-visible changes

---

## 21. Best Practices

The principles above, condensed:

- **The spec is the contract.** Firmware, software and simulator all follow the protocol specification — never each other's quirks. Drift is found by testing both ends against the document.
- **Design for the unplugged cable.** Every device interaction times out, fails with context, and leaves both ends in a known state. Hardware tools earn trust in failure, not in success.
- **Keep the core pure.** I/O at the edges, logic in the middle, tests everywhere the logic is. If it needs hardware to test, keep shrinking it until it doesn't.
- **Own your abstractions, rent your dependencies.** Project-owned interfaces around everything that churns — transports, SDKs, frameworks. The rewrite you avoid is the one behind an interface.
- **Boring beats clever.** Standard language style, standard project layout, standard CLI conventions, minimal dependencies. Novelty is spent where AURIORA is actually novel — the hardware — not in the tooling around it.
- **Scriptable first.** Everything works non-interactively with honest exit codes and stable structured output. Humans get a friendly interface; automation gets a reliable one; both drive the same core.
- **Data outlives tools.** Open formats, versioned schemas, explicit units, provenance attached, atomic writes. The calibration file must be readable long after the tool that wrote it is history.
- **Make it observable.** Version everywhere, wire-level tracing one flag away, diagnostics as a feature. The tool that can explain itself gets debugged in minutes, not meetings.

---

*AURIORA Software Style Guide 0.1.0 — complements the AURIORA Engineering Standard. Licensed under CC BY-SA 4.0.*
