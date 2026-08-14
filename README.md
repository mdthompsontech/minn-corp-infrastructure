# Minn-Corp Infrastructure

Real infrastructure documentation from a self-hosted homelab — a production VPS running behind Cloudflare Zero Trust, an AI-agent administration model, and the operational discipline built up around both. This is the actual working document, sanitized for publication, not something rewritten after the fact for a portfolio.

## What this is

I run a small production stack (VPS, reverse proxy, Zero Trust tunnel, fail2ban, an AI coding agent used for host administration) and maintain a versioned baseline document as the single source of truth for its state. This repo is that document, with network identifiers replaced for safety — everything else, including the mistakes, is unedited.

**[SANITIZED_BASELINE_v17.md](./SANITIZED_BASELINE_v17.md)** — the full infrastructure reference: architecture, security policy, operational procedures, open items, and a maintained incident history.

## Why the incident history is the part worth reading

Section 13 of the baseline is a running list of things that went wrong, in the order they happened, each one tied to the rule it produced. A few examples:

- A brute-force protection tool was installed pointed at the wrong firewall backend — it would have reported healthy while blocking nothing. Caught before it mattered.
- An alerting script silently failed to deliver for weeks because a credential file's format changed. Every observable signal said it was healthy. It wasn't.
- A destructive package removal was aborted mid-transaction after reading the full dependency list, which showed it would have taken a critical security tool down with it.
- A versioned document got edited in place by an AI agent, producing two files that both claimed to be the same version with different content. Caught, repaired, and turned into a standing rule against ever doing it again.

I'd rather show the near-misses and how they were caught than present a record with no mistakes in it. A clean changelog doesn't prove anything; a record of things that almost went wrong, and the process that caught them, does.

## How this is maintained

Three-document system:
- **RULES** — a static preamble defining how an AI assistant should behave when helping administer this system: verification discipline, formatting conventions, security guardrails.
- **MASTER_BASELINE** — the living state of the system. Versioned; each revision is a complete standalone replacement, never a patch, so there is always one unambiguous current truth. The sanitized copy in this repo is that document.
- **SESSION_LOG** — an append-only record of what happened in each working session and why.

## Sanitization

Public IP addresses use [RFC 5737](https://www.rfc-editor.org/rfc/rfc5737) / [RFC 3849](https://www.rfc-editor.org/rfc/rfc3849) documentation ranges. The domain is rewritten to `.example`. Account emails, cloud provider instance IDs, and access-policy identifiers are redacted or replaced. Internal (RFC 1918) network design, hostnames, configuration, and the incident history are unaltered — the internal architecture is the part worth showing, and none of it is reachable from outside the network it describes.

## Background

Built while transitioning from 20+ years in hospital operations toward IT infrastructure, cloud, and security roles. The self-hosted stack — and the discipline of documenting it accurately under real change — is the practical work behind that transition.
