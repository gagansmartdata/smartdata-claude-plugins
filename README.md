# Smartdata Plugins

Claude Code plugins published by [Gagan](https://gagansmartdata.github.io/).

Each agent is packaged as its own plugin. Add the marketplace once, then install only the agents you want — they are independent and nothing pulls in anything else.

## Install

Add the marketplace:

```
/plugin marketplace add gagansmartdata/smartdata-claude-plugins
```

Then install whichever agents you want:

```
/plugin install code-improver@smartdata-plugins
/plugin install technical-writer@smartdata-plugins
/plugin install security-auditor@smartdata-plugins
/plugin install compliance-auditor@smartdata-plugins
```

Or browse and pick from a menu:

```
/plugin
```

If an install summary says `Run /reload-plugins to activate.`, run that. Otherwise you're done — no restart needed.

To remove one:

```
/plugin uninstall technical-writer@smartdata-plugins
```

> The marketplace is added by **repo** name (`smartdata-claude-plugins`), but plugins are installed by **marketplace** name (`smartdata-plugins`) — that's the `name` field in `.claude-plugin/marketplace.json`.

## Available agents

| Plugin | What it does | Writes files? | Model |
| :--- | :--- | :--- | :--- |
| [`code-improver`](plugins/code-improver) | Reviews code and suggests improvements for readability, performance, and best practices | No — read-only | `sonnet` |
| [`technical-writer`](plugins/technical-writer) | Writes user guides, getting-started tutorials, SDK and admin docs, and troubleshooting content | Yes | `haiku` |
| [`security-auditor`](plugins/security-auditor) | Finds exploitable weaknesses in code and config — injection, broken auth, crypto, secrets, SSRF | No — read-only | `inherit` |
| [`compliance-auditor`](plugins/compliance-auditor) | Audits code against GDPR, HIPAA, PCI DSS, SOC 2, and ISO 27001 and grades the project per framework | No — read-only | `inherit` |

---

### `code-improver`

A read-only code reviewer. It scans your files and returns concrete suggestions across three axes — readability, performance, and best practices — and for each one it explains the problem, quotes the code as it stands, and gives you a rewritten version.

**It cannot edit your code.** The agent is defined with `tools: Read, Glob, Grep` and nothing else, so there is no mechanism by which it can write to disk. Every suggestion comes back as a diff-able code block that you apply, or don't.

Use it:

```
Use the code-improver agent to review src/auth.ts
```

Or @-mention it to guarantee it runs rather than relying on auto-delegation:

```
@"code-improver (agent)" review the files I changed on this branch
```

<details>
<summary>Example of what comes back</summary>

### 1. Membership test over a growing array — Medium · Performance

`src/importer.ts:84`

**Issue:** `seen` is an array and `seen.includes(id)` rescans it on every iteration, so importing an *n*-row file does roughly *n²/2* comparisons. At 500 rows it's invisible; at 50,000 rows this loop alone takes tens of seconds.

**Current:**

```ts
const seen = [];
for (const row of rows) {
  if (seen.includes(row.id)) continue;
  seen.push(row.id);
  await save(row);
}
```

**Improved:**

```ts
const seen = new Set();
for (const row of rows) {
  if (seen.has(row.id)) continue;
  seen.add(row.id);
  await save(row);
}
```

**Why this is better:** `Set.has` is constant-time, making the loop linear. Drop-in replacement — the only behavioral difference is that `Set` uses SameValueZero equality, which matches `includes` for the string IDs used here.

</details>

**What it looks for**

| Axis | Examples |
| :--- | :--- |
| Readability | Misleading names, functions doing several jobs, deep nesting that flattens with early returns, duplicated logic, dead code, comments that restate the code |
| Performance | Repeated work inside loops, accidental O(n²) scans, N+1 queries, sequential `await`s over independent work, unnecessary copying of large structures |
| Best practices | Swallowed exceptions, unhandled rejections, missing input validation, resource leaks, mutation of arguments, hardcoded secrets, SQL built by concatenation |

It's also told what *not* to report: settled style preferences the formatter already owns, and padding to make the list look thorough. If a file is in good shape it says so.

**Configuration**

| Setting | Value |
| :--- | :--- |
| `tools` | `Read, Glob, Grep` |
| `model` | `sonnet` |

Fork and edit [`plugins/code-improver/agents/code-improver.md`](plugins/code-improver/agents/code-improver.md) to change either — the frontmatter is the whole configuration surface.

---

### `technical-writer`

A technical writer for documentation people read start to finish: getting-started guides, task-based user guides, SDK and administrator manuals, troubleshooting pages, and FAQs. It audits the existing docs for gaps, clarity problems, and stale content before writing, and organizes what it produces around what a reader is trying to accomplish rather than around how the software is structured.

**This one writes to disk** — `tools: Read, Write, Edit, Glob, Grep, WebFetch, WebSearch`. Point it at a docs directory rather than a whole repository.

Use it:

```
Use the technical-writer agent to write a getting-started guide for this CLI
```

Or @-mention it:

```
@"technical-writer (agent)" audit docs/ and tell me what's missing or out of date
```

**What it covers**

| Area | Examples |
| :--- | :--- |
| Doc types | Developer docs, end-user guides, administrator manuals, SDK docs, integration guides, troubleshooting |
| Writing technique | Progressive disclosure, task-based writing, minimalist approach, structured authoring, single sourcing |
| Content standards | Style guides, voice and tone, formatting rules, terminology consistency, accessibility, SEO |
| Information architecture | Logical organization, navigation, categorization, cross-references, search, user pathways |
| Visual communication | Diagrams, flowcharts, architecture diagrams, screenshots and annotations |
| Automation | Doc generation, snippet extraction, changelog automation, link checking, version sync |

It targets a readability score above 60 with verified technical accuracy, and is told to check claims against the code rather than assume them.

**Configuration**

| Setting | Value |
| :--- | :--- |
| `tools` | `Read, Write, Edit, Glob, Grep, WebFetch, WebSearch` |
| `model` | `haiku` |

Fork and edit [`plugins/technical-writer/agents/technical-writer.md`](plugins/technical-writer/agents/technical-writer.md) to change either.

---

### `security-auditor`

A read-only application security auditor. It finds **exploitable weaknesses** in code and configuration, traces how each one is reached, and ranks them by real exploitability.

**It cannot edit your code.** Defined with `tools: Read, Grep, Glob` and nothing else — no Write, no Edit, no Bash. It reports; you remediate.

Use it:

```
Use the security-auditor agent to review the auth and session handling in src/
```

Or @-mention it:

```
@"security-auditor (agent)" audit the changes on this branch for security issues
```

**Reachability first.** What separates a useful security report from a linter dump is whether anyone can actually reach the bug. The agent finds the entry point, finds the sink, then greps the call graph to establish that untrusted input connects them — checking middleware, decorators, guards, and base classes, since the missing check is usually not at the call site. When it can't establish that path the finding is labeled **Unconfirmed**, with a note on what couldn't be verified, rather than quietly promoted.

**What it looks for**

| Class | Examples |
| :--- | :--- |
| Injection | SQL and NoSQL by concatenation, command execution, LDAP and XPath, template injection, ORM raw-fragment escape hatches |
| Authentication | Bypassable checks, non-constant-time comparison, weak hashing, tokens from non-cryptographic randomness, unverified JWT signatures, `alg: none` |
| Authorization | Missing checks on data-access paths, IDOR, privilege escalation, mass assignment setting role or owner, tenant isolation failures |
| Cryptography | MD5/SHA-1/DES/RC4, ECB mode, reused IVs, hardcoded keys, `Math.random` for security values, disabled cert verification, secrets in source or CI |
| Data exposure | Sensitive values in logs, stack traces returned to clients, telemetry and URLs, debug endpoints and source maps shipped |
| Request handling | SSRF including cloud metadata and redirect bypasses, path traversal, unrestricted upload, XXE, unsafe deserialization, CSRF, prototype pollution |
| Configuration | Wildcard CORS with credentials, default credentials, over-permissive IAM and storage in IaC, privileged containers, secrets as build args |
| Logic and concurrency | TOCTOU races, non-atomic check-then-act on balances and one-time tokens, missing idempotency, no rate limiting on auth |
| Dependencies | Unpinned versions, install-time scripts, abandoned packages in security-critical paths, typosquat-adjacent names |

**Severity** runs Critical (remotely reachable unauthenticated → RCE, auth bypass, bulk data access) through High, Medium, Low, and Unconfirmed — ranked by exploitability and impact, never by how hard the fix is.

Each finding carries the weakness class, `path:line`, the code as it appears, and an **attack path**: the entry point, the parameter an attacker controls, how it reaches the sink, and what they get — named concretely, not described in the abstract. Then the fix, including whether a leaked credential should be treated as burned. One root cause across many call sites is reported once with a count, not filed per site.

**Know what it can't do.** It reads only what you point it at — no live network, running host, IdP, or cloud console, so runtime and infrastructure-enforced controls are out of reach. With no `WebFetch` or `WebSearch` it can't check a dependency against an advisory database, so it flags dependencies as *worth verifying* rather than asserting CVEs. And it is not a penetration test: static reading finds weakness classes, it doesn't prove exploitation.

**Configuration**

| Setting | Value |
| :--- | :--- |
| `tools` | `Read, Grep, Glob` |
| `model` | `inherit` |

Reachability tracing is capability-sensitive — following a call graph across files is where smaller models start guessing — so `inherit` is deliberate. Fork and edit [`plugins/security-auditor/agents/security-auditor.md`](plugins/security-auditor/agents/security-auditor.md) to change either.

---

### `compliance-auditor`

Audits your codebase against **GDPR**, **HIPAA**, **PCI DSS**, **SOC 2**, and **ISO 27001/27002**, reports findings in priority order with file-and-line evidence, and **grades the project per framework** with an explicit coverage figure.

**It cannot edit your code.** Defined with `tools: Read, Grep, Glob` and nothing else — no Write, no Edit, no Bash. It audits and reports; remediation is yours.

Use it:

```
Use the compliance-auditor agent to audit this repo and give me a compliance score
```

Or scope it to a framework or requirement:

```
@"compliance-auditor (agent)" audit src/payments/ against PCI DSS Req. 3 and 4
```

**It works out which frameworks apply** by finding the data first — schemas, migrations, models, payloads. If there's no cardholder data in the codebase it reports PCI DSS as *not applicable* and says why, rather than filing twenty hollow findings. That's the usual failure mode of a compliance checklist.

**The scorecard**

| Framework | Grade | Score | Satisfied | Partial | Failed | Not assessable | Coverage |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| GDPR | B | 84 | 16 | 4 | 2 | 5 | 81% |
| PCI DSS | F | 71 | 12 | 3 | 4 | 3 | 86% |
| SOC 2 | — | 78 (indicative) | 7 | 2 | 2 | 19 | 37% |

```
score    = (satisfied + 0.5 × partial) / (satisfied + partial + failed) × 100
coverage = (satisfied + partial + failed) / total controls in scope × 100
```

`Not assessable from code` controls are excluded from the score and reported as coverage — **never counted as passes**. Grades run A (90–100) to F (below 60), with two overrides: **any open P0 caps that framework at F** regardless of the arithmetic, and **coverage under 40% suppresses the letter grade** entirely, leaving a score marked *indicative only*. The overall project marking is the **lowest** applicable framework grade, not the average — compliance isn't a mean.

**Findings** run P0 (active violation with reportable exposure — prohibited CVV storage, regulated data in logs, no encryption in transit) through P1 (a required control with no implementation found), P2 (partial or inconsistent), P3 (hardening and evidence quality), plus Observations for what can't be confirmed from code. Each finding carries the control reference, `path:line`, the code as it appears, the requirement it fails, and whether the fix is drop-in or needs a migration or a policy decision.

Every report ends with a mandatory **Not assessable from code** section — the controls you must verify elsewhere. That section is what stops the scorecard being read as a clean bill of health.

**Know what it can't do.** It is not an attestation and not legal advice — no auditor or regulator will accept it as evidence. Much of SOC 2 and ISO 27001 is organizational (policies, training, vendor management, physical security) and has no code footprint, so expect low coverage there. It reads only the files you point it at — no live network, host, IdP, or cloud console — so controls implemented in infrastructure outside the repo will read as absent. And with no `WebFetch` or `WebSearch` it can't check a dependency against an advisory database, so spot-check version-specific claims and control citations.

**`compliance-auditor` vs `security-auditor`** — the two are deliberately scoped so they don't compete for delegation:

| | `compliance-auditor` | `security-auditor` |
| :--- | :--- | :--- |
| Question it answers | "Do we satisfy the controls, and how do we score?" | "Where are we exploitable, and what do we fix first?" |
| Unit of work | The framework control | The attack path |
| Organized around | GDPR, HIPAA, PCI DSS, SOC 2, ISO 27001 | Weakness classes and reachability |
| Distinctive output | Per-framework scorecard with coverage and a project grade | Severity-ranked findings with traced attack paths |
| Reach for it when | Preparing for an audit, reporting posture upward | Hardening a system, reviewing a diff, triaging risk |

Each agent's `description` names the other and says when to hand off, so auto-delegation picks by intent rather than by keyword overlap. Both are read-only, so running both is safe.

**Configuration**

| Setting | Value |
| :--- | :--- |
| `tools` | `Read, Grep, Glob` |
| `model` | `inherit` |

Control mapping and citation accuracy degrade noticeably on smaller models, so `inherit` is deliberate. Fork and edit [`plugins/compliance-auditor/agents/compliance-auditor.md`](plugins/compliance-auditor/agents/compliance-auditor.md) to change either.

## Repository layout

```
smartdata-claude-plugins/
├── .claude-plugin/
│   └── marketplace.json               # marketplace manifest — lists the plugins below
├── plugins/
│   ├── code-improver/
│   │   ├── .claude-plugin/
│   │   │   └── plugin.json            # plugin manifest
│   │   ├── agents/
│   │   │   └── code-improver.md       # the subagent definition
│   │   └── README.md
│   ├── technical-writer/
│   │   ├── .claude-plugin/
│   │   │   └── plugin.json
│   │   ├── agents/
│   │   │   └── technical-writer.md
│   │   └── README.md
│   ├── security-auditor/
│   │   ├── .claude-plugin/
│   │   │   └── plugin.json
│   │   ├── agents/
│   │   │   └── security-auditor.md
│   │   └── README.md
│   └── compliance-auditor/
│       ├── .claude-plugin/
│       │   └── plugin.json
│       ├── agents/
│       │   └── compliance-auditor.md
│       └── README.md
├── LICENSE
└── README.md
```

`agents/` sits at the **plugin root**, not inside `.claude-plugin/`. Only `plugin.json` goes in there — this is the most common way a plugin ends up loading with none of its components.

One plugin per agent is deliberate: it's what lets someone install `technical-writer` without also getting `code-improver`.

## Contributing

To add a plugin:

1. Create `plugins/<name>/.claude-plugin/plugin.json` with `name`, `description`, and `version`.
2. Add components at the plugin root: `agents/`, `skills/<name>/SKILL.md`, `hooks/hooks.json`, `.mcp.json`.
3. Append an entry to the `plugins` array in `.claude-plugin/marketplace.json` with `name` and `source: "./plugins/<name>"`.
4. Add a `README.md` at the plugin root and a section in this file.
5. Validate both manifests before opening a PR:

   ```bash
   claude plugin validate ./plugins/<name>
   claude plugin validate .
   ```

Test locally without installing:

```bash
claude --plugin-dir ./plugins/technical-writer
```

Run `/reload-plugins` to pick up edits mid-session.

## Versioning

Installed copies only update when the plugin's `version` field changes. Bump it in **both** `plugins/<name>/.claude-plugin/plugin.json` and the matching entry in `.claude-plugin/marketplace.json`, in the same commit. Leaving the two out of sync is the usual reason a published fix doesn't reach anyone.

## License

[MIT](LICENSE) © Smartdata Enterprises India Limited
