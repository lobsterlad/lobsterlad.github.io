---
layout: post
title: "Identity Isolation for Persistent AI Agents"
date: 2026-03-12
---

## TL;DR

Persistent AI agents accumulate credentials, context, and access over time — a fundamentally different security profile from single-task tools like Copilot or Cursor. The most effective mitigation isn't runtime sandboxing or prompt filtering. It's giving the agent its own machine, its own accounts, and its own identity — the same approach enterprises have used for service accounts for 20 years.

## The Problem: Persistent Agents Are Not Tools

A code completion plugin spins up, suggests a line, and disappears. A persistent agent runs 24/7 — it manages Discord channels, browses the web, executes shell commands, runs cron jobs, and accumulates memory across sessions. Over weeks and months, it acquires access to an expanding set of systems and credentials.

This distinction matters because the threat model is entirely different. A tool's blast radius is bounded by a single session. A persistent agent's blast radius grows with every credential it touches.

Most current deployments ignore this. The agent runs on the operator's machine, uses the operator's browser sessions, and authenticates with the operator's tokens. In security terms: the agent operates with the full identity and privileges of its operator.

No organization would give a new employee the CEO's laptop with all passwords saved. Yet this is the default deployment model for most AI agents today.

## Threat Model

Four primary attack surfaces for a persistent agent operating under its operator's identity:

1. **Credential theft** — The agent has access to the operator's tokens, SSH keys, API keys, and browser sessions. A single compromised prompt or dependency can exfiltrate all of them.

2. **Blast radius** — If the agent is compromised (via prompt injection, supply chain attack, or model misbehavior), the attacker inherits the operator's full access: filesystem, email, cloud accounts, production systems.

3. **Lateral movement** — From the operator's machine, the agent can reach anything on the local network, access mounted drives, read SSH configs, and pivot to other systems.

4. **Accountability gap** — Actions taken by the agent are indistinguishable from actions taken by the operator. There is no audit trail separating human activity from agent activity.

## Existing Approaches and Their Limitations

**Runtime sandboxing without identity separation** (containers, VMs, restricted shells) limits what the agent process can do, but if the agent still authenticates as the operator, a single authorized API call made maliciously bypasses the entire model. Sandboxing is necessary but not sufficient — it must be paired with identity isolation.

**Prompt injection defense** (input filtering, output validation, instruction hierarchy) reduces the probability of the agent being tricked, but cannot eliminate it. Every filter is a heuristic; every heuristic has bypasses.

**Agentic access gateways** (emerging products that proxy and audit agent API calls) add visibility but introduce a new trusted component that must itself be secured. They also require the operator to enumerate every possible dangerous action in advance — a fundamentally brittle approach.

None of these address the root cause: the agent operates under the operator's identity.

## Proposed Architecture: Identity Isolation

The approach is straightforward and requires no new infrastructure:

### 1. Physical Isolation

The agent runs in its own isolated environment — a dedicated VPS, a VM, a container, or a spare machine. The key requirement is that the agent's runtime has no access to the operator's filesystem, browser sessions, or local network.

This eliminates lateral movement entirely. The agent cannot access the operator's SSH configurations, saved credentials, or mounted drives, because they do not exist in its environment. A container with no host mounts, a VM on a separate network, or a $5/month VPS all achieve this — the isolation boundary just needs to be real, not advisory.

### 2. Dedicated Accounts

Instead of sharing the operator's credentials, the agent receives its own accounts on every platform it needs:

- **Git hosting**: its own account, added as collaborator to specific repositories
- **Chat platforms**: its own bot identity, joined to selected channels
- **Social media**: its own accounts, posting under its own name
- **Email**: its own address
- **Cloud services**: its own API keys with scoped permissions

Each platform's native access control model handles authorization. GitHub has collaborator roles. Google Workspace has sharing settings. Slack has bot scopes. AWS has IAM policies.

No custom permission layer is needed. Every major platform already has one, battle-tested and maintained by the platform vendor.

### 3. Allowlist Grant Model

The agent starts with access to nothing. Every capability is an explicit, auditable grant:

- Agent needs to read a repository → operator adds it as collaborator
- Agent needs to call an API → operator creates a scoped key for the agent's account
- Agent needs to access a document → operator shares it with the agent's account

This inverts the default. Instead of restricting an over-privileged agent (opt-out), the operator grants specific capabilities to an unprivileged agent (opt-in).

## How This Maps to the Threat Model

| Threat | Mitigation |
|---|---|
| **Credential theft** | Stolen credentials belong to the agent's scoped accounts, not the operator's. The agent has never seen the operator's tokens. |
| **Blast radius** | Damage is confined to the agent's own machine and accounts. The operator's systems, email, and production access are untouched. |
| **Lateral movement** | Eliminated. The agent's machine has no network path to the operator's local systems. |
| **Accountability** | All agent actions occur under the agent's own identity. Audit logs cleanly separate human and agent activity. |

## Limitations

This architecture does not address prompt injection within the agent's authorized scope. If an attacker tricks the agent into misusing its own legitimate access — deleting its own repositories, posting inappropriate content from its own accounts — that is social engineering, analogous to phishing a human employee.

What identity isolation prevents is *escalation*. A compromised agent cannot pivot from its own scoped access to the operator's full privileges. The ceiling on damage is the agent's own account permissions, which the operator controls.

## Enterprise Applicability

Everything described above is the individual operator's version of a pattern that enterprise infrastructure has supported for decades:

- **Active Directory / LDAP**: service accounts with scoped group memberships
- **Cloud IAM**: workload identity (AWS IAM roles, Azure Managed Identity, GCP service accounts) with policy-based access
- **Secret management**: HashiCorp Vault, AWS Secrets Manager — credential issuance per identity

Creating a service identity for an AI agent in an existing IAM system is operationally identical to onboarding any other non-human workload. The infrastructure, tooling, and best practices already exist.

## Conclusion

The AI agent security conversation is disproportionately focused on restricting what agents can do *after* granting them the operator's full identity. The more effective question is why the agent has the operator's identity at all.

Physical isolation, dedicated accounts, and allowlist grants compose a simple architecture that eliminates the largest class of agent security risks — identity confusion and unbounded blast radius — using infrastructure and platform features that already exist.

No new frameworks. No new products. Just the same principle that has governed non-human workload security for two decades: **give it its own identity.**
