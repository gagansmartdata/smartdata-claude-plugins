# Compliance Auditor

A read-only Claude Code subagent that audits your codebase against **GDPR**, **HIPAA**, **PCI DSS**, **SOC 2**, and **ISO 27001/27002**, reports findings in priority order with file-and-line evidence, and grades the project per framework.

It has no editing tools — `tools: Read, Grep, Glob` and nothing else — so it cannot change your files or run commands. It audits and reports; every remediation is yours to apply.

## Install

```
/plugin marketplace add gagansmartdata/smartdata-claude-plugins
/plugin install compliance-auditor@smartdata-plugins
```

Run `/reload-plugins` if the install summary asks you to.

## Usage

Natural language:

```
Use the compliance-auditor agent to audit this repo and give me a compliance score
```

Guaranteed invocation via @-mention:

```
@"compliance-auditor (agent)" check us against GDPR and PCI DSS only
```

Scope it to a framework, a requirement, or a directory when you want depth over breadth:

```
@"compliance-auditor (agent)" audit src/payments/ against PCI DSS Req. 3 and 4
```

Given nothing specific it audits the working directory and works out which frameworks apply from the data it finds.

## How it decides what applies

It looks for the data first — schemas, migrations, models, DTOs, API payloads — and establishes what regulated data the system actually handles. A framework only gets audited if its data class is present.

That matters, because the failure mode of a compliance checklist is manufacturing findings against a framework that doesn't apply. If there's no cardholder data in the codebase, this agent reports PCI DSS as **not applicable** and says why, instead of filing twenty hollow findings.

## What comes back

**1. Scope** — directories and file types reviewed, what was skipped, which frameworks apply and which don't.

**2. Scorecard** — a grade per applicable framework, plus an overall project marking:

| Framework | Grade | Score | Satisfied | Partial | Failed | Not assessable | Coverage |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| GDPR | B | 84 | 16 | 4 | 2 | 5 | 81% |
| PCI DSS | F | 71 | 12 | 3 | 4 | 3 | 86% |
| SOC 2 | — | 78 (indicative) | 7 | 2 | 2 | 19 | 37% |

**3. Findings**, P0 first, each carrying the framework and control reference, a `path:file.ext:LINE` location, the code as it appears, the specific requirement it fails and when exposure occurs, and remediation — including whether the fix is drop-in or needs a data migration, a backfill, or a policy decision.

**4. Not assessable from code** — mandatory section listing controls you must verify elsewhere. This is what stops the scorecard being read as a clean bill of health.

**5. Positive findings** — controls genuinely well implemented, so you know what not to disturb.

## Priority levels

Findings are classified by regulatory exposure, not by how hard they are to fix.

| Priority | Meaning |
| :--- | :--- |
| **P0 — Critical** | Active, demonstrable violation with reportable or penalty-bearing exposure — prohibited storage of CVV or track data, regulated data in plaintext or in logs, no encryption in transit, hardcoded production credentials |
| **P1 — High** | A required control has no implementation in the code reviewed — no audit trail for PHI access, no erasure path for personal data, missing authorization on a regulated-data endpoint |
| **P2 — Medium** | The control exists but is partial or inconsistent — soft-delete-only erasure, audit logs missing the actor, retention set in one store but not another |
| **P3 — Low** | Hardening or evidence-quality gap; the control works but would be hard to demonstrate to an auditor |
| **Observation** | Not a finding — context, or a risk that can't be confirmed from code |

## How the grade is calculated

Every control assessed is classified Satisfied, Partial, Failed, or **Not assessable from code**. Then:

```
score    = (satisfied + 0.5 × partial) / (satisfied + partial + failed) × 100
coverage = (satisfied + partial + failed) / total controls in scope × 100
```

`Not assessable` controls are excluded from the score and surfaced as coverage — **never counted as passes**. A score that quietly treats what code can't show as compliant is worse than no score at all.

| Grade | Score |
| :--- | :--- |
| A | 90–100 |
| B | 80–89 |
| C | 70–79 |
| D | 60–69 |
| F | < 60 |

Two overrides apply after scoring:

- **Any open P0 caps that framework at F**, whatever the arithmetic says. One prohibited-storage finding isn't offset by ninety controls done well.
- **Coverage below 40% suppresses the letter grade** — you get a score marked *indicative only* with the coverage figure, and no letter.

The overall project marking is the **lowest** applicable framework grade, not the average. Compliance isn't a mean.

## Scope and limits

Read this before you put a grade from this agent in front of anyone:

- **It is not an attestation.** This is an internal, code-scoped assessment that helps you prepare for an audit and prioritize remediation. No auditor, certification body, or regulator will accept it as evidence, and it is not legal advice.
- **Much of SOC 2 and ISO 27001 has no code footprint.** Policies, training, vendor management, physical security, risk process — a code auditor cannot see any of it. Expect low coverage on those two frameworks, and read the *Not assessable* section as the real output there.
- **It reads only what you point it at.** No live network, no running host, no IdP, no cloud console. Controls implemented in infrastructure you don't keep in the repo will read as absent — the agent is instructed to say "no implementation found in the code reviewed" rather than "the control is missing," but you still have to supply that context.
- **No web access.** It has no `WebFetch` or `WebSearch`, so it cannot check a dependency against an advisory database or verify current framework text. Control citations and version-specific claims are worth spot-checking.

## `compliance-auditor` vs `security-auditor`

This marketplace also ships [`security-auditor`](../security-auditor). The two are deliberately scoped so they don't compete:

| | `compliance-auditor` | `security-auditor` |
| :--- | :--- | :--- |
| Question it answers | "Do we satisfy the controls, and how do we score?" | "Where are we exploitable, and what do we fix first?" |
| Unit of work | The framework control | The attack path |
| Organized around | GDPR, HIPAA, PCI DSS, SOC 2, ISO 27001 | Weakness classes and reachability |
| Distinctive output | Per-framework scorecard with coverage and a project grade | Severity-ranked findings with traced attack paths |
| Reach for it when | Preparing for an audit or certification, reporting posture upward | Hardening a system, reviewing a diff, triaging risk |

This agent is instructed **not** to build attack paths or threat models, and to hand off if the request is about exploitability with no framework driver. Where a control failure is also an exploitable bug, it reports the control failure with its citation and leaves exploitability in the impact line.

Both are read-only, so running both is safe and gives you two lenses on the same code.

## Configuration

| Setting | Value | Why |
| :--- | :--- | :--- |
| `tools` | `Read, Grep, Glob` | Read-only by construction — no Write, Edit, or Bash |
| `model` | `inherit` | Runs on the session's model, so a full audit gets your strongest one rather than a fixed cheap default |

Both live in the frontmatter of [`agents/compliance-auditor.md`](agents/compliance-auditor.md). Compliance reasoning is capability-sensitive — control mapping and citation accuracy degrade noticeably on smaller models — so pin this to a specific model only if you have a reason to.

Change either setting there, then bump `version` in `.claude-plugin/plugin.json` and in the marketplace entry so installed copies pick up the update.

## License

[MIT](../../LICENSE) © Smartdata Enterprises India Limited
