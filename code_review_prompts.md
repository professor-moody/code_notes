# Offensive Tool Repo Review — Prompt Kit

## How to Use

Pick the tier that matches the tool's operational context. Most tools fall into **Tier 2**.

| Tier | Use When | Examples |
|------|----------|----------|
| **Tier 1 — Covert/Operational** | Tool runs on target, stealth matters, detection = mission failure | loaders, implants, UDRL work, sleep masking, anything touching the CI/CD obfuscation pipeline output |
| **Tier 2 — Assessment & Exploitation** | Tool runs with authorization to find/exploit vulns, correctness and reliability matter most | enum, bloodhound-mcp, OffSec tooling, NTLM relay tooling, things generally without opsec concerns. |
| **Tier 3 — Infrastructure & Pipeline** | Build systems, lab automation, support tooling | GitLab CI/CD pipeline, Ludus range configs, etc |
