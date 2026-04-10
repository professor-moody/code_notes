# Offensive Tool Repo Review — Prompt Kit

## How to Use

Pick the tier that matches the tool's operational context. Most tools fall into **Tier 2**.

| Tier | Use When | Examples |
|------|----------|----------|
| **Tier 1 — Covert/Operational** | Tool runs on target, stealth matters, detection = mission failure | loaders, implants, UDRL work, sleep masking, anything touching the CI/CD obfuscation pipeline output |
| **Tier 2 — Assessment & Exploitation** | Tool runs with authorization to find/exploit vulns, correctness and reliability matter most | enum, bloodhound-mcp, OffSec tooling, NTLM relay tooling, things generally without opsec concerns. |
| **Tier 3 — Infrastructure & Pipeline** | Build systems, lab automation, support tooling | GitLab CI/CD pipeline, Ludus range configs, etc |

---

# Tier 1 — Covert/Operational Tool Review

## Objective

Perform a thorough, read-only review of this repository. This is **offensive security tooling** where the threat model is not "can an attacker exploit our app" but rather "will this tool fail us during an engagement, leak our presence, mishandle captured data, or break against real-world target variability." Prioritize **operational reliability, OPSEC, correctness of exploit/protocol logic, and build integrity** — then architecture, maintainability, and test quality.

---

## Phase 0 — Orientation & Baseline

Before reviewing code:

1. **Map the repo structure.** Identify languages, build systems, entrypoints (CLI bins, API servers, agent entrypoints, module interfaces). Understand what runs on the operator's machine vs. what touches target environments vs. what runs in CI/CD.
2. **Inventory dependencies.** Check pinning hygiene and lockfile presence. Flag any dependency that phones home, has telemetry enabled by default, or could introduce supply chain risk into the operator's build chain. Note vendored vs. fetched deps — offensive tools should minimize external fetch-at-build risk.
3. **Run existing tests.** Execute the full suite and record pass/fail/skip. Existing failures are baseline noise — document them separately. If tests can't run due to missing target infra or fixtures, document why rather than skipping.
4. **Review CI/CD pipeline config.** Check for credential exposure in pipeline variables, artifact handling (are compiled tools/payloads stored securely?), build reproducibility, and whether obfuscation or modification steps (PE timestamps, hash modification, signature stripping) are consistently applied.

---

## Phase 1 — OPSEC Review

The highest-priority class of findings. Any of these can burn an engagement.

### 1.1 Operator Fingerprint Leakage
- Hardcoded paths, usernames, hostnames, internal IPs, or org-identifying strings in source, compiled output, or config defaults.
- User-agent strings, HTTP headers, TLS client fingerprints, or protocol handshake values that are unusual or attributable.
- DNS lookups, NTP queries, update checks, or any outbound network calls the tool makes that aren't directed at the target.
- Logging or debug output that writes operator-side paths, environment variables, or system info to disk or network.
- Build artifacts that embed build-host metadata (Go binary build paths, .NET assembly metadata, PE rich headers, compilation timestamps that leak timezone).

### 1.2 Target Artifact Residue
- Files, registry keys, event log entries, scheduled tasks, services, or named pipes created on target — are they documented and cleaned up on exit?
- Does the tool have a clean teardown path? What happens on SIGINT/SIGTERM/unexpected disconnect — does it leave artifacts behind?
- Temporary files on target: are they written to predictable paths, do they have conspicuous names, are they cleaned on exit?
- Memory residue: does the tool handle sensitive data (creds, tokens, keys) in memory carefully, or does it leave cleartext scattered across heap allocations?

### 1.3 Detection Surface
- Network behavior: connection patterns, beaconing regularity, protocol deviations that an NDR/IDS would flag.
- Host behavior: process lineage, command-line arguments visible in ETW/Sysmon, handles and named objects, unusual API call sequences.
- Are stealth/verbosity levels configurable? Can the operator dial back noise without sacrificing core functionality?
- Any use of well-signatured techniques or known-burned IOCs that should be flagged as detection-likely.

---

## Phase 2 — Operational Reliability

The tool needs to work when you're in a target environment with one shot. Review for robustness under adversarial and degraded conditions.

### 2.1 Target Environment Variability
- Version sensitivity: does the tool assume a specific version of the target software? How does it behave against older/newer versions or non-default configurations?
- Protocol correctness: are protocol implementations (SMB, Kerberos, LDAP, HTTP APIs, gRPC, MCP, REST endpoints) robust against quirks, non-standard responses, and partial data?
- Encoding and locale: does the tool handle non-ASCII hostnames, paths, usernames, Unicode in credentials, different line endings, non-English locale responses?
- Permission and access variations: does the tool degrade gracefully when it has fewer privileges than expected, or does it crash/hang?

### 2.2 Error Handling Under Operational Conditions
- Network instability: how does the tool handle connection drops, timeouts, throttling, TCP resets mid-operation? Can it resume or does it lose state?
- Partial success: if the tool is enumerating/exfiltrating multiple targets and one fails, does it continue or abort entirely?
- Unexpected responses: malformed JSON, unexpected HTTP status codes, empty responses, target-side rate limiting, WAF/proxy interference.
- Do errors propagate clearly to the operator, or are they swallowed silently, leaving the operator unaware that a module failed?

### 2.3 Credential & Loot Handling (Operator Side)
- How are captured credentials, tokens, keys, and loot stored on the operator's system? Encrypted at rest or plaintext in `/tmp`?
- Credential chain-loading: when the tool uses a captured credential to pivot, is the chain tracked and auditable for the engagement report?
- Are credentials passed safely between modules/components, or exposed in CLI args (visible in `ps`), environment variables, or log output?
- Cleanup: can the operator securely purge captured data after an engagement?

### 2.4 Concurrency & State
- If the tool runs parallel operations (multi-target enumeration, concurrent API calls, agent tasking), are shared state and output streams handled correctly?
- Race conditions on file writes (multiple modules writing to the same output file/DB).
- Resource exhaustion: unbounded thread/goroutine spawning, connection pool starvation, or memory growth on large target environments.

---

## Phase 3 — Architecture & Design

### 3.1 Module Boundaries & Extensibility
- Are modules (target-specific modules, protocol handlers, output formatters) cleanly separated so new targets/techniques can be added without modifying core logic?
- Shared state or tight coupling between modules that would make adding a new module fragile.
- Plugin or module discovery: if the tool has a modular architecture, is the registration/discovery pattern robust and well-documented?

### 3.2 Data Flow & Output Integrity
- Trace the data path from target interaction through to operator-visible output (CLI, file, dashboard, database). Where can data be corrupted, dropped, duplicated, or misattributed?
- Output format consistency: if the tool generates structured output (JSON, JSONL, database records), are schemas consistent across modules?
- Are results attributable — can the operator trace a finding back to the specific target, technique, and credential chain that produced it?

### 3.3 Configuration & Defaults
- Are defaults safe and OPSEC-conscious (e.g., no default verbose logging, no default cleartext output, no default beaconing at obvious intervals)?
- Config loading precedence and validation — can a typo in config silently cause the tool to run against the wrong target or with wrong credentials?
- Are target connection parameters (URLs, ports, paths) validated before use, or does a bad config surface as a cryptic runtime error?

### 3.4 Dependency & Build Chain Integrity
- Are build outputs reproducible? Could a compromised build host inject something into the pipeline output?
- For the CI/CD pipeline: are offensive tool builds isolated from each other? Could one compromised upstream dependency affect all tools in the pipeline?
- License exposure: any dependency with a license that requires disclosure or distribution of source, which would be problematic for client-delivered tools?

---

## Phase 4 — Test Quality & Coverage Gaps

### 4.1 Core Logic Coverage
- Are the tool's core invariants tested (e.g., "credential extraction produces valid, usable credentials," "enumeration discovers all accessible targets," "module X correctly implements protocol Y")?
- Are protocol/API interactions tested against realistic responses, including edge cases and error responses?

### 4.2 Failure Mode Testing
- Are degraded conditions tested — connection drops, auth failures, partial responses, rate limiting, unexpected target versions?
- Does the test suite verify that OPSEC-sensitive operations (cleanup, artifact removal, safe credential handling) actually work?

### 4.3 Test Realism & Hygiene
- Are tests using realistic mocks/fixtures, or are they testing against idealized responses that don't reflect production target environments?
- Tests that depend on external services, network access, or specific lab infrastructure without clear documentation.
- Flaky tests, order-dependent tests, tests asserting on timing or non-deterministic values.

### 4.4 Regression Anchors
- When a target environment quirk or protocol edge case was previously discovered and fixed, is there a test locking that fix in place?

---

## Phase 5 — Maintainability (Defect-Risk Lens)

Only flag when it creates operational risk or slows safe development:

- Dead code that creates false confidence in capabilities or coverage.
- Copy-paste protocol logic where a fix in one path would be missed in another.
- Misleading comments or docs that could cause a contributor to break OPSEC or operational behavior.
- Magic strings/numbers — especially target-specific constants (default ports, API paths, version strings) that should be configurable but are hardcoded.
- Build/release footguns: manual steps in the release process that could be forgotten and produce broken or un-obfuscated artifacts.

---

## Deliverable Format

### 1. Executive Summary
Two to three sentences: overall operational readiness, highest-risk finding category, and whether the tool is in a "deploy on engagement," "fix before next engagement," or "stop and remediate" posture.

### 2. Findings (ordered by severity)

| Tier | Label | Definition |
|------|-------|------------|
| P0 | **Critical** | OPSEC failure (operator exposure, attribution risk), data loss/corruption of captured loot, or tool behavior that could damage a target environment outside scope. |
| P1 | **High** | Operational failure under realistic conditions — tool crashes, hangs, or produces incorrect results against plausible target configurations. |
| P2 | **Medium** | Incorrect behavior requiring specific conditions to trigger, missing error handling that degrades operator experience, or gaps in cleanup/teardown. |
| P3 | **Low** | Robustness or maintainability issue that increases defect probability over time but won't bite you on the next engagement. |
| Info | **Informational** | Observation, pattern note, or enhancement suggestion. |

For each finding:
- **Title:** One-line description.
- **Severity:** P0–P3 / Info.
- **Location:** File(s), line(s), function/module names.
- **Operational Impact:** What goes wrong during an engagement — concrete scenario.
- **Evidence:** Code reference, test output, or trace.
- **Pre-existing vs. New:** If determinable.
- **Suggested Remediation:** Brief, actionable direction.

### 3. Notable Strengths
Patterns, abstractions, or coverage done well. What to preserve and build on.

### 4. Residual Risk Areas
Modules, protocol implementations, or operational flows not fully validated. Where confidence is low.

### 5. Dependency & Build Chain Summary
Total deps, any with telemetry/phone-home behavior, any unmaintained, any with problematic licenses for offensive tool distribution.

### 6. Next Inspection Targets
Hotspots worth a deeper focused pass, with rationale.

---

## Constraints

- **Read-only.** No source modifications. Drift or broken contracts reported as findings.
- **Evidence over opinion.** Every finding above Info needs a concrete scenario and code reference.
- **Separate baseline from new.** Pre-existing test failures in their own section.
- **Operational context matters.** Judge the code by how it performs during an engagement against real targets, not by abstract best practices for production SaaS.

---

# Tier 2 — Assessment & Exploitation Tool Review

## Objective

Perform a thorough, read-only review of this repository. This is an **offensive security assessment/exploitation tool** — it runs with authorization against target environments. The primary concerns are **correctness of security logic, reliability against real-world target variability, output accuracy, and operator experience** — not stealth. A bug here means missed findings, false results, wasted engagement time, or a crashed tool at the worst possible moment.

---

## Phase 0 — Orientation & Baseline

1. **Map the repo structure.** Identify entrypoints, module/plugin architecture, CLI interface, output mechanisms. Understand what the tool claims to do — enumerate, exploit, extract, scan — and how the operator interacts with it.
2. **Inventory dependencies.** Pinning hygiene, lockfile presence. Flag dependencies that are unmaintained, abandoned, or have known bugs that would affect protocol/API interactions. Note anything that would make installation painful in a typical pentest environment (Kali, macOS, Windows, containers).
3. **Run existing tests.** Record pass/fail/skip. Baseline failures documented separately. Note any tests that require live target infrastructure to run and whether that's documented.
4. **Identify the target surface.** List every target technology, protocol, or API the tool interacts with (e.g., Ollama API, ChromaDB REST, MCP protocol, SMB/LDAP/Kerberos, Docker socket, LangChain configs). This inventory drives the rest of the review.

---

## Phase 1 — Correctness of Security & Exploitation Logic

This is where assessment tools live or die. A tool that misidentifies a vulnerability or silently skips an attack path is worse than having no tool.

### 1.1 Technique/Exploit Implementation Accuracy
- For each attack technique or assessment check the tool implements: is the logic correct against the relevant specification, advisory, or known-good reference implementation?
- Are there assumptions baked into the exploit logic that would cause silent failure against certain target configurations (e.g., assuming default ports, default credentials, specific API versions, specific response formats)?
- Version-specific behavior: does the tool account for differences across target versions, or does it implement against one version and hope?
- Are there known variants or edge cases of the technique that the tool doesn't cover but should? (e.g., covering ESC1 but missing a related misconfiguration path.)

### 1.2 Enumeration & Discovery Completeness
- Does the enumeration logic actually find everything it should, or can targets/configurations/credentials be missed due to pagination handling, filtering logic, or early termination?
- Are there code paths where a partial failure during enumeration (one API call fails, one host unreachable) causes the tool to silently return incomplete results without telling the operator?
- Permission boundary handling: when the tool has limited access, does it report what it *could* see vs. what it *couldn't*, or does it conflate "not found" with "access denied"?

### 1.3 Protocol & API Implementation
- For each protocol or API the tool speaks: is the implementation correct against real-world behavior, not just the spec? (Specs lie. Implementations vary.)
- Authentication handling: does the tool correctly implement auth flows against the target (API keys, tokens, Kerberos tickets, NTLM, certificates)? Does it handle auth failures, token expiry, re-auth gracefully?
- Request construction: are requests well-formed? Headers, content-types, encoding, parameter formatting — would a slightly non-standard target reject them?
- Response parsing: is the tool parsing responses robustly, or will an unexpected field, missing field, different data type, or extra whitespace cause a crash or silent misparse?

### 1.4 Credential & Data Extraction Accuracy
- When the tool extracts credentials, configs, secrets, or other sensitive data from a target: is the extraction logic correct? Will it produce valid, usable output?
- Format handling: are extracted credentials correctly parsed and formatted for downstream use (e.g., correct hash format for cracking, correct token format for reuse)?
- Is extraction complete, or could data be truncated, mangled, or partially captured?

---

## Phase 2 — Operational Reliability

The tool runs during a time-boxed engagement. It needs to work the first time and tell you clearly when something goes wrong.

### 2.1 Target Environment Variability
- What target versions/configurations has the tool been tested against vs. what it claims to support?
- How does the tool behave against targets that are slightly different than expected — newer version, non-default config, different OS, containerized vs. bare metal, behind a proxy/load balancer?
- Encoding and locale: non-ASCII values in target data (hostnames, usernames, database contents, file paths, configuration values).
- Platform-specific behavior: if the tool runs on multiple OS platforms, are there path handling, networking, or subprocess differences that could cause inconsistent behavior?

### 2.2 Error Handling & Operator Feedback
- **This is critical for assessment tools.** When something fails, the operator needs to know *what* failed, *why*, and *whether partial results are trustworthy.*
- Are errors surfaced to the operator with enough context to act on them, or are they generic ("connection error," "failed," "null")?
- Silent failures: are there code paths where a module/check fails but the tool continues and reports success or completeness without indicating the gap?
- Can the operator distinguish between "target is not vulnerable" and "the check couldn't run"? This distinction is essential for accurate reporting.
- Timeout handling: are all target-facing operations bounded? What happens when a target is slow but not dead?

### 2.3 Output Integrity
- Is tool output (to CLI, files, database) accurate, consistent, and complete?
- Can output from different modules or scan passes be correlated and deduplicated?
- If the tool writes structured output (JSON, CSV, database), is the schema consistent across modules and target types?
- Concurrency: if the tool runs parallel operations, can output be interleaved, corrupted, or misattributed?
- Is there enough metadata in the output for the operator to reproduce or verify a finding (target, timestamp, technique, credentials used, exact request/response if relevant)?

### 2.4 State Management & Resumability
- If the tool crashes or is interrupted mid-run, is progress lost entirely or can it resume?
- For long-running assessments (large target environments, many hosts), is state tracked so the operator can monitor progress?
- Are results persisted incrementally or only at the end?

---

## Phase 3 — Architecture & Extensibility

### 3.1 Module/Plugin Architecture
- Can new targets, techniques, or checks be added without modifying core logic? How much boilerplate does a new module require?
- Is the module interface well-defined — clear inputs (target connection, credentials, config), clear outputs (findings, extracted data, errors)?
- Module isolation: does a bug or crash in one module affect others?

### 3.2 Data Flow
- Trace the path from target interaction → parsing → internal representation → output. Where can data be lost, corrupted, or misrepresented?
- Are internal data structures appropriate for the domain (e.g., credential types, finding severity, target metadata)?

### 3.3 Configuration & Defaults
- Are defaults sensible for common engagement scenarios?
- Can config errors (wrong target URL, bad credentials, typo in module name) be caught early and reported clearly, or do they surface as cryptic runtime failures?
- Is the configuration documented enough that a teammate could run the tool from the README without guessing?

### 3.4 Dependencies & Portability
- Does the tool install and run cleanly on typical pentest platforms (Kali, macOS, common Docker base images)?
- Dependency conflicts with other common pentest tools?
- Binary size and deployment weight (especially for tools where "zero dependency, small binary" is a design goal).

---

## Phase 4 — Test Quality

### 4.1 Core Logic Coverage
- Are the tool's assessment checks and exploit logic tested against realistic responses — including edge cases, error responses, and version variations?
- Are protocol/API interactions tested with mocks that reflect real target behavior, not just idealized happy-path responses?

### 4.2 False Positive / False Negative Testing
- Are there tests that verify the tool does NOT flag something that isn't vulnerable (false positive)?
- Are there tests that verify the tool DOES catch known-vulnerable configurations (false negative)?
- This is the most important testing dimension for assessment tools.

### 4.3 Failure Mode Coverage
- Connection failures, auth failures, partial responses, unexpected versions, malformed data from targets.
- Does the test suite verify that error messages are accurate and helpful?

### 4.4 Regression Tests
- When a target-specific quirk or parsing edge case was fixed, is there a test anchoring that fix?

---

## Phase 5 — Maintainability (Defect-Risk Only)

- Dead modules or checks that appear functional but haven't been updated for current target versions.
- Copy-paste protocol logic across modules where a fix in one would be missed in others.
- Hardcoded target-specific constants (default ports, API paths, version strings) that should be configurable or at least centralized.
- Misleading module names, descriptions, or help text that would cause an operator to misunderstand what a check does or doesn't cover.

---

## Deliverable Format

### 1. Executive Summary
Overall assessment of the tool's reliability and correctness posture. Would you trust this tool's output on an engagement tomorrow?

### 2. Findings (ordered by severity)

| Tier | Label | Definition |
|------|-------|------------|
| P0 | **Critical** | Produces incorrect security findings (false negatives on real vulns, false positives that waste client time), corrupts/loses extracted data, or could cause unintended impact on target outside assessment scope. |
| P1 | **High** | Fails against realistic target configurations, silently returns incomplete results, or provides misleading operator feedback that would affect engagement quality. |
| P2 | **Medium** | Incorrect behavior under specific conditions, missing error handling that degrades operator experience, output formatting issues that complicate reporting. |
| P3 | **Low** | Robustness or maintainability issue — won't bite you this engagement but accumulates risk. |
| Info | **Informational** | Enhancement suggestion, pattern observation, coverage gap worth noting. |

Per finding: **Title, Severity, Location, Operational Impact** (what goes wrong during an engagement), **Evidence, Pre-existing vs. New, Suggested Remediation.**

### 3. Notable Strengths

### 4. Residual Risk Areas

### 5. Target Coverage Matrix
Quick table: each target/technique the tool claims to support, current test coverage level, and confidence that the implementation is correct against current target versions.

### 6. Next Inspection Targets

---

## Constraints
- **Read-only.** No modifications.
- **Evidence over opinion.** Concrete scenario and code reference for every finding above Info.
- **Baseline separation.** Pre-existing failures in their own section.
- **Judge by engagement reality.** The standard is "does this tool produce correct, reliable results against real targets" — not abstract code quality.

---

# Tier 3 — Infrastructure & Pipeline Tool Review

## Objective

Review build systems, CI/CD pipelines, lab automation, and internal support tooling. The concerns are **build integrity, reproducibility, secret safety in pipeline context, and operational reliability of the infrastructure itself.**

---

## Phase 0 — Orientation & Baseline
Same as other tiers: map structure, run tests, understand entrypoints.

---

## Phase 1 — Build Integrity & Supply Chain

### 1.1 Reproducibility
- Are builds deterministic? Same input → same output?
- Are all dependencies pinned to exact versions with lockfiles?
- Is there drift between what CI builds and what a developer builds locally?

### 1.2 Pipeline Security
- Secrets handling: are pipeline variables/secrets scoped correctly? Could a malicious MR/PR exfiltrate secrets via build logs or artifact output?
- Are pipeline steps isolated from each other where needed, or could one compromised tool's build step affect another?
- Artifact integrity: are built tools signed, hashed, or otherwise verifiable after build?

### 1.3 Supply Chain Risk
- Dependencies that fetch remote resources at build time (install scripts, curl-pipe-bash, remote container base images without digest pinning).
- Any dependency with telemetry, analytics, or phone-home behavior that could be included in built offensive tools.
- For the obfuscation/modification pipeline: are transformations (PE timestamp randomization, hash modification) applied consistently and verifiably?

---

## Phase 2 — Operational Reliability

### 2.1 Lab & Range Automation
- Are deployments idempotent? Can a range be torn down and rebuilt cleanly?
- Error handling in provisioning: what happens when Packer/Ansible/Terraform fails mid-run?
- Resource management: are VMs/containers cleaned up reliably, or do orphaned resources accumulate?

### 2.2 Configuration Management
- Are lab configs version-controlled and parameterized, or do they embed environment-specific hardcoded values?
- Can a teammate stand up the same environment from the repo without tribal knowledge?

---

## Phase 3–5
Architecture, tests, and maintainability — follow the same structure as Tier 2 but scoped to infrastructure concerns.

---

## Deliverable Format

Same severity tiers as Tier 2, but P0 reframed:

| Tier | Label | Definition |
|------|-------|------------|
| P0 | **Critical** | Pipeline compromise risk (secret leak, artifact tampering), or builds producing tools with unintended telemetry/attribution. |
| P1 | **High** | Build failures under realistic conditions, non-reproducible outputs, or lab automation that leaves orphaned resources. |
| P2 | **Medium** | Configuration drift, inconsistent obfuscation/modification steps, or missing error handling in provisioning. |
| P3 | **Low** | Maintainability or documentation gaps that slow safe pipeline changes. |
| Info | **Informational** | Enhancement suggestion or pattern observation. |

Everything else follows the same template: Executive Summary, Findings, Notable Strengths, Residual Risk Areas, Dependency Summary, Next Inspection Targets.
