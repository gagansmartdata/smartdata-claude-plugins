# Code Improver

A read-only Claude Code subagent that reviews code and suggests improvements for **readability**, **performance**, and **best practices**.

It has no editing tools — `tools: Read, Glob, Grep` and nothing else — so it cannot change your files. Every fix comes back as a code block you apply yourself.

## Install

```
/plugin marketplace add smartdataenterprises/smartdata-plugins
/plugin install code-improver@smartdata-plugins
```

Run `/reload-plugins` if the install summary asks you to.

## Usage

Natural language:

```
Use the code-improver agent to review src/auth.ts
```

Guaranteed invocation via @-mention:

```
@"code-improver (agent)" review the files I changed on this branch
```

Point it at a file, a set of files, a directory, or a pattern. Given nothing specific, it picks source files in the working directory — skipping lockfiles, minified bundles, generated code, and vendored dependencies — and tells you which it chose.

## What you get back

A short summary of what was reviewed, then findings ordered high to low severity. Each one carries:

- **Severity and category** — High / Medium / Low, and which of the three axes it falls under
- **Location** — `path/to/file.ext:LINE`
- **Issue** — the concrete input or condition under which it bites, not a general principle
- **Current** — the code exactly as it appears in your file
- **Improved** — the rewritten version
- **Why this is better** — including any behavior change or trade-off, and whether it is a drop-in replacement

Severity means:

| Level | Meaning |
| :--- | :--- |
| High | A bug, security issue, data-loss risk, or resource leak |
| Medium | Real maintainability or performance cost, worth fixing soon |
| Low | Small clarity win, safe to defer |

## Configuration

| Setting | Value | Why |
| :--- | :--- | :--- |
| `tools` | `Read, Glob, Grep` | Read-only by construction — no Write, Edit, or Bash |
| `model` | `sonnet` | Fast and cheap enough to run over a whole directory |

Both live in the frontmatter of [`agents/code-improver.md`](agents/code-improver.md). Change either there, then bump `version` in `.claude-plugin/plugin.json` and in the marketplace entry so installed copies pick up the update.

## Design notes

The agent is instructed to:

- Read surrounding context and grep for usages before calling anything dead, redundant, or safe to change
- Quote current code verbatim rather than reconstructing it from memory
- Match the conventions already in the codebase instead of imposing its own
- Skip settled style preferences — formatter output, quote style, import ordering — unless they are inconsistent within one file
- Say a file is fine when it is fine, rather than padding the list to look thorough

## License

[MIT](../../LICENSE) © Smartdata Enterprises India Limited
