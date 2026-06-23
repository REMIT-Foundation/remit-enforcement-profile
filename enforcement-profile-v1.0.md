# REMIT Enforcement Profile
## Reference Architectures for Non-Bypassable Agent Control

**Companion to the REMIT Core Specification and Agent Passport v1.0**

Version 1.0 | Published June 2026 | REMIT Foundation

Licensed under Creative Commons Attribution 4.0 International (CC BY 4.0).

---

## Foreword

REMIT requires that every agent action pass through an interception layer. Agent Passport requires that layer to be non-bypassable and capable of validating signed capability tokens. This document describes recognised enforcement architectures, their bypass resistance properties, and conformance criteria for implementers.

It does not mandate a single technology. It defines what each architecture must prove to claim REMIT + Passport compliance.

**The REMIT Foundation**
June 2026

---

## Contents

1. Introduction
2. Enforcement Principles
3. Reference Architectures
4. Conformance Criteria
5. Bypass Resistance Checklist
6. Combining Architectures

---

## 1. Introduction

### 1.1 Purpose

The enforcement layer is the trusted boundary between an untrusted agent runtime and the tool layer. It functions as a **semantic execution firewall**: it inspects tool calls (not IP packets) and allows or denies based on capability tokens, passport manifests and REMIT verdicts.

### 1.2 Relationship to Other Specifications

| Specification | Role |
|---|---|
| REMIT Core §5.3, §6 | Requires interception; defines integration modes |
| Agent Passport §9 | Defines token validation and non-bypassability requirements |
| ATMS §3.3 | Tool layer threat surface |

---

## 2. Enforcement Principles

Every conformant enforcement architecture must satisfy:

1. **Mandatory mediation** — no direct tool or network access from the agent runtime.
2. **Deny by default** — unmapped capabilities and endpoints are blocked.
3. **Dual validation** — passport/token check and REMIT verdict (order: passport deny short-circuits).
4. **Isolation matches policy** — forbidden systems are unreachable, not only blocked in software.
5. **Independent audit** — the agent cannot write to or delete enforcement audit records.

---

## 3. Reference Architectures

### 3.1 SDK Broker (Highest Control)

**Pattern:** Agent code registers tools only through a REMIT/Passport SDK. Unregistered tools cannot be invoked.

```
Agent code → SDK.intercept(tool, params) → validate → execute broker
```

| Property | Rating |
|---|---|
| Bypass resistance | High (if CI/CD gate enforces SDK usage) |
| Code change | Required |
| Passport support | Native token validation in SDK |
| Best for | Greenfield agents, internal platforms |

**Required controls:** build-time scan for uninstrumented tool calls (REMIT §6.4 CI/CD Gate).

---

### 3.2 Network Proxy / API Gateway

**Pattern:** All outbound HTTP/gRPC from the agent passes through Envoy, Kong, or similar. Policy compiled from passport manifest.

```
Agent → egress proxy → validate JWT capability token → upstream API
```

| Property | Rating |
|---|---|
| Bypass resistance | Medium (network policy must block direct egress) |
| Code change | None |
| Passport support | JWT validation filter, route table from manifest |
| Best for | Legacy agents, language-agnostic deployments |

**Required controls:** container/network policy denying direct egress except to proxy; no ambient credentials in agent container.

---

### 3.3 MCP Server Gate

**Pattern:** Agent connects to a single REMIT-compliant MCP server. All tools are MCP tools behind the gate.

```
Agent (MCP client) → REMIT MCP server → validate → tool handlers
```

| Property | Rating |
|---|---|
| Bypass resistance | Medium–High (depends on MCP being sole tool interface) |
| Code change | Config only |
| Passport support | Token presented at MCP session init |
| Best for | Claude, LangChain, AutoGen MCP-native stacks |

**Required controls:** disable non-MCP tool providers in agent configuration.

---

### 3.4 Service Mesh Sidecar

**Pattern:** Sidecar intercepts all traffic from the agent workload. Policy driven by passport manifest CRDs or xDS.

```
Agent pod → sidecar → validate → service broker
```

| Property | Rating |
|---|---|
| Bypass resistance | High (with strict pod security and network policies) |
| Code change | Minimal |
| Passport support | Wasm filter for token validation |
| Best for | Kubernetes-native platforms at scale |

**Required controls:** `NetworkPolicy` matching manifest `denies`; no host networking; read-only root filesystem.

---

### 3.5 Sandboxed Runtime (gVisor / Firecracker / WASM)

**Pattern:** Agent runs in a hardened sandbox. Only brokered syscalls and network paths available.

```
Sandboxed agent → narrow syscall surface → enforcement broker
```

| Property | Rating |
|---|---|
| Bypass resistance | High |
| Code change | Varies |
| Passport support | Broker validates tokens before syscall forward |
| Best for | High assurance, multi-tenant agent hosting |

**Required controls:** seccomp, no shell, no raw sockets; attestation for Level 4 assurance.

---

## 4. Conformance Criteria

An implementation claims **REMIT Enforcement Profile v1.0 conformance** when it demonstrates:

| Criterion | Evidence |
|---|---|
| E1 Interception | Every tool call in a test suite passes through the enforcement layer |
| E2 Deny by default | Unlisted tool/endpoint returns deny without execution |
| E3 Passport validation | Invalid, expired and revoked tokens are rejected |
| E4 Manifest denies | Explicit deny rules block before tool layer |
| E5 REMIT integration | Allow/Flag/Block verdicts recorded per §5.3 |
| E6 Audit independence | Agent cannot modify enforcement audit store |
| E7 Isolation | Forbidden network ranges unreachable from agent (network test) |

Conformance is self-declared for v1.0. Third-party assessment may be required for REMIT Level 4 (High Assurance).

---

## 5. Bypass Resistance Checklist

Before production deployment, operators must verify:

- [ ] Agent container has no credentials for tools except those held by the broker.
- [ ] Agent cannot open raw TCP/UDP to arbitrary hosts (egress denied except proxy).
- [ ] SSH, shell and code-exec tools are absent or unregistered unless explicitly in manifest.
- [ ] Sub-agents receive attenuated tokens only (delegation test).
- [ ] Revoked token is rejected within 60 seconds (connected) or at next CRL sync (air-gapped).
- [ ] Audit log entries include `passport_id` and `token_id`.
- [ ] REMIT Block triggers operator alert within 60 seconds.

---

## 6. Combining Architectures

High-assurance deployments should combine layers:

| Layer | Example |
|---|---|
| CI/CD | Fail build on uninstrumented tool calls |
| Runtime SDK | Intercept and validate tokens |
| Network | Egress proxy + deny rules from manifest |
| Sandbox | gVisor + seccomp |
| Authority | Local passport authority with HSM |

Example stack: **CI/CD gate + MCP gate + local authority + network policies from manifest denies**.

This implements defence in depth: bypassing one layer should not collapse the entire control plane.

---

## Contact

spec@remitframework.org
