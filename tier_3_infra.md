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
