# Technical Writer

A Claude Code subagent that writes and maintains technical documentation for people: getting-started guides, task-based user guides, SDK and administrator docs, troubleshooting pages, and FAQs.

It audits what exists before writing — looking for content gaps, clarity problems, and stale material — then produces documentation organized around what a reader is trying to accomplish rather than around how the software is built.

## Install

```
/plugin marketplace add gagansmartdata/smartdata-claude-plugins
/plugin install technical-writer@smartdata-plugins
```

Run `/reload-plugins` if the install summary asks you to.

## Usage

Natural language:

```
Use the technical-writer agent to write a getting-started guide for this CLI
```

Guaranteed invocation via @-mention:

```
@"technical-writer (agent)" audit docs/ and tell me what's missing or out of date
```

Things it handles well:

- Getting-started and quick-start guides for a new tool or service
- Task-based user guides — "how do I do X" rather than "here is what X does"
- Rewriting existing docs that are accurate but hard to follow
- Troubleshooting pages and FAQs built from real failure modes
- Administrator and installation manuals
- Information architecture — how a docs site should be organized and navigated
- Style guides and terminology consistency across a docs set

## What it covers

| Area | Includes |
| :--- | :--- |
| Doc types | Developer docs, end-user guides, administrator manuals, API references, SDK docs, integration guides, troubleshooting |
| Writing technique | Progressive disclosure, task-based writing, minimalist approach, structured authoring, single sourcing, localization-ready copy |
| Content standards | Style guides, voice and tone, formatting rules, terminology consistency, accessibility, SEO |
| Information architecture | Logical organization, navigation, categorization, cross-references, search, user pathways |
| Visual communication | Diagrams, flowcharts, architecture diagrams, screenshots and annotations, interactive elements |
| Review | Technical accuracy, clarity, completeness, consistency, accessibility testing |
| Automation | Doc generation, snippet extraction, changelog automation, link checking, build integration, version sync |

It targets a readability score above 60 and verified technical accuracy, and it is told to check claims against the code rather than assume.

## Configuration

| Setting | Value | Why |
| :--- | :--- | :--- |
| `tools` | `Read, Write, Edit, Glob, Grep, WebFetch, WebSearch` | Needs write access to create and revise doc files; web access to check conventions and upstream references |
| `model` | `haiku` | Documentation is high-volume prose work once the product is understood |

Both live in the frontmatter of [`agents/technical-writer.md`](agents/technical-writer.md).

> **Note:** this agent **can write to disk** — it has `Write` and `Edit`. That is deliberate: it produces documentation files. Point it at a docs directory rather than turning it loose on a whole repository.

Change either setting there, then bump `version` in `.claude-plugin/plugin.json` and in the marketplace entry so installed copies pick up the update.

## License

[MIT](../../LICENSE) © Smartdata Enterprises India Limited
