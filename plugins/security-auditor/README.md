# Security Auditor

A read-only Claude Code subagent that finds **exploitable weaknesses** in source code and configuration, traces how each one is reached, and ranks them by real exploitability.

It has no editing tools — `tools: Read, Grep, Glob` and nothing else — so it cannot change your files or run commands. It reports; you remediate.

## Install

```
/plugin marketplace add gagansmartdata/smartdata-claude-plugins
/plugin install security-auditor@smartdata-plugins
```

Run `/reload-plugins` if the install summary asks you to.

## Usage

Natural language:

```
Use the security-auditor agent to review the auth and session handling in src/
```

Guaranteed invocation via @-mention:

```
@"security-auditor (agent)" audit the changes on this branch for security issues
```

Point it at a repository, a directory, a service, a diff, or a single file. Given nothing specific it audits the working directory and tells you what it chose.

## Reachability first

The thing that separates a useful security report from a linter dump is whether anyone can actually reach the bug. This agent is instructed to trace the path before reporting: find the entry point, find the sink, then grep the call graph to establish that untrusted input actually connects them — checking middleware, decorators, guards, and base classes, since the missing check is often not at the call site.

When it can't establish that path, the finding is labeled **Unconfirmed** with a note on exactly what couldn't be verified. It doesn't get quietly promoted to a vulnerability.

## What it looks for

| Class | Examples |
| :--- | :--- |
| Injection | SQL and NoSQL by concatenation, command execution with shell metacharacters, LDAP and XPath, template injection, ORM raw-fragment escape hatches |
| Authentication | Bypassable checks, non-constant-time comparison, weak hashing, tokens from non-cryptographic randomness, no expiry or revocation, unverified JWT signatures, `alg: none` |
| Authorization | Missing checks on data-access paths, IDOR, horizontal and vertical escalation, mass assignment setting role or owner, tenant isolation failures |
| Cryptography | MD5/SHA-1/DES/RC4, ECB mode, static or reused IVs, hardcoded keys, `Math.random` for security values, disabled certificate verification, secrets in source or CI config |
| Data exposure | Sensitive values in logs, stack traces returned to clients, telemetry and URLs, debug endpoints and source maps shipped, over-broad API responses |
| Request handling | SSRF including cloud metadata and redirect bypasses, path traversal, unrestricted upload, XXE, unsafe deserialization, open redirect, CSRF, prototype pollution |
| Configuration | Wildcard CORS with credentials, missing security headers, default credentials, over-permissive IAM and storage in IaC, privileged or root containers, secrets as build args |
| Logic and concurrency | TOCTOU races, non-atomic check-then-act on balances and one-time tokens, missing idempotency on destructive operations, unbounded resource use, no rate limiting on auth |
| Dependencies | Unpinned versions, install-time scripts, plaintext fetches, abandoned packages in security-critical paths, typosquat-adjacent names |

## Severity

Ranked by exploitability and impact, never by how hard the fix is.

| Level | Meaning |
| :--- | :--- |
| **Critical** | Remotely reachable unauthenticated, yielding code execution, auth bypass, or bulk sensitive-data access |
| **High** | Exploitable by an authenticated or low-privilege attacker for escalation, cross-tenant access, or data disclosure |
| **Medium** | Needs unusual conditions or a chain with another weakness, or yields limited impact — but real and demonstrable |
| **Low** | Hardening and defense-in-depth; no demonstrable exploit path alone |
| **Unconfirmed** | Usually-a-vulnerability pattern where reachability couldn't be established from the code available |

## What comes back

Each finding carries the weakness class, a `path:file.ext:LINE` location, the code as it appears, and an **attack path** — the entry point, the parameter an attacker controls, how it reaches the sink, and what they get, named concretely rather than described in the abstract. Then the fix, including whether it's drop-in or needs a migration, a credential rotation, or a design decision, and whether a leaked credential should be treated as burned.

Findings sharing one root cause are reported once with the representative line and a full count, rather than filed per call site. The report closes with positive findings and a mandatory section on what the audit could not cover.

## `security-auditor` vs `compliance-auditor`

This marketplace also ships [`compliance-auditor`](../compliance-auditor). They are deliberately scoped so they don't compete:

| | `security-auditor` | `compliance-auditor` |
| :--- | :--- | :--- |
| Question it answers | "Where are we exploitable, and what do we fix first?" | "Do we satisfy the controls, and how do we score?" |
| Unit of work | The attack path | The framework control |
| Organized around | Weakness classes and reachability | GDPR, HIPAA, PCI DSS, SOC 2, ISO 27001 |
| Distinctive output | Severity-ranked findings with traced attack paths | Per-framework scorecard with coverage and a project grade |
| Reach for it when | Hardening a system, reviewing a diff, triaging risk | Preparing for an audit or certification, reporting posture upward |

This agent is instructed **not** to produce compliance scorecards or map findings to control identifiers, and to hand off if the request is framework-driven. It may note that a finding carries regulatory weight — plaintext cardholder data is a PCI problem too — but the finding stays framed by exploitability.

Both are read-only, so running both is safe and gives you two lenses on the same code.

## Scope and limits

- **It reads what you point it at.** Source, configuration, IaC, CI files, dependency manifests. It cannot scan a network, probe a running host, execute code, query your IdP, or read a cloud console — so runtime behavior and infrastructure-enforced controls are outside its reach.
- **No web access.** It has no `WebFetch` or `WebSearch`, so it cannot check a dependency against an advisory database. It is instructed to flag dependencies as *worth verifying* rather than asserting specific CVEs — treat any version-specific claim as a lead.
- **It is not a penetration test.** Static reading finds weakness classes; it does not prove exploitation. High-value findings still deserve confirmation in a real environment.

## Configuration

| Setting | Value | Why |
| :--- | :--- | :--- |
| `tools` | `Read, Grep, Glob` | Read-only by construction — no Write, Edit, or Bash |
| `model` | `inherit` | Runs on the session's model, so a deep audit gets your strongest one rather than a fixed cheap default |

Both live in the frontmatter of [`agents/security-auditor.md`](agents/security-auditor.md). Reachability tracing is capability-sensitive — following a call graph across files is where smaller models start guessing — so pin this to a specific model only if you have a reason to.

Change either setting there, then bump `version` in `.claude-plugin/plugin.json` and in the marketplace entry so installed copies pick up the update.

## License

[MIT](../../LICENSE) © Smartdata Enterprises India Limited
