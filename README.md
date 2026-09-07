# Smartdata Plugins

Claude Code plugins published by [Smartdata Enterprises](https://github.com/smartdataenterprises).

## Install

```
/plugin marketplace add gagansmartdata/smartdata-claude-plugins
/plugin install code-improver@smartdata-claude-plugins
```

If the install summary says `Run /reload-plugins to activate.`, run that. Otherwise you're done — no restart needed.

## Plugins

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

## Repository layout

```
smartdata-claude-plugins/
├── .claude-plugin/
│   └── marketplace.json          # marketplace manifest — lists the plugins below
├── plugins/
│   └── code-improver/
│       ├── .claude-plugin/
│       │   └── plugin.json       # plugin manifest
│       ├── agents/
│       │   └── code-improver.md  # the subagent definition
│       └── README.md
├── LICENSE
└── README.md
```

`agents/` sits at the **plugin root**, not inside `.claude-plugin/`. Only `plugin.json` goes in there — this is the most common way a plugin ends up loading with none of its components.

## Contributing

To add a plugin:

1. Create `plugins/<name>/.claude-plugin/plugin.json` with `name`, `description`, and `version`.
2. Add components at the plugin root: `agents/`, `skills/<name>/SKILL.md`, `hooks/hooks.json`, `.mcp.json`.
3. Append an entry to the `plugins` array in `.claude-plugin/marketplace.json` with `name` and `source: "./plugins/<name>"`.
4. Validate both manifests before opening a PR:

   ```bash
   claude plugin validate ./plugins/<name>
   claude plugin validate .
   ```

Test locally without installing:

```bash
claude --plugin-dir ./plugins/code-improver
```

Run `/reload-plugins` to pick up edits mid-session.

## Versioning

Installed copies only update when the plugin's `version` field changes. Bump it in **both** `plugins/<name>/.claude-plugin/plugin.json` and the matching entry in `.claude-plugin/marketplace.json`, in the same commit. Leaving the two out of sync is the usual reason a published fix doesn't reach anyone.

## License

[MIT](LICENSE) © Smartdata Enterprises India Limited
