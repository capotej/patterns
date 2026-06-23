---
id: P003
name: using-mise
date: 2026-06-21
author: Julio Capote
used_in: ['github.com/capotej/abbey', 'github.com/capotej/pi-mise', 'github.com/boldblackai/harness']
---

# Using mise

A single `mise.toml` in the project root declares every tool and language version the project needs.

## When to use

- Any project that needs specific versions of languages (Ruby, Node, Python, Bun, Go) or CLI tools (shellcheck, hadolint, actionlint, jj, etc.).
- Any project where developers and agents should run the exact same tool versions without manual setup.
- Any project used inside containerized agent environments (harness) where `mise activate` ensures the agent picks up the right tools automatically.

## How it works

### The mise.toml file

Place a `mise.toml` in the project root. It contains a single `[tools]` section listing every tool and its pinned version:

```toml
[tools]
ruby = "3.4.1"
bun = "1.2.4"
```

```toml
[tools]
jj = "0.41.0"
"github:koalaman/shellcheck" = "v0.11.0"
"github:hadolint/hadolint" = "v2.14.0"
"github:rhysd/actionlint" = "v1.7.7"
```

### Version pinning rules

1. **Always pin to an exact version.** Never use ranges (`>= 3.4`), `latest`, or leave versions unpinned. The whole point is reproducibility.
2. **Exception for `latest`:** Only acceptable for tools where you always want the bleeding edge and accept the non-reproducibility trade-off (e.g., `jj = "latest"` in pi-mise — acceptable because jj updates frequently and the extension always wants the newest).
3. **GitHub-sourced tools** use the `"github:owner/repo" = "vX.Y.Z"` syntax. This lets mise install tools that aren't in its built-in registry. The version string must match the GitHub release tag exactly (including the `v` prefix if the release uses one).

### Tool categories

mise.toml typically declares tools from three categories:

| Category | Examples | Syntax |
|---|---|---|
| Language runtimes | ruby, node, python, bun, go | `ruby = "3.4.1"` |
| CLI tools (mise registry) | jj, terraform, node | `jj = "0.41.0"` |
| CLI tools (GitHub releases) | shellcheck, hadolint, actionlint | `"github:koalaman/shellcheck" = "v0.11.0"` |

### Relationship to legacy version files

mise reads many legacy version files (`.ruby-version`, `.node-version`, `.python-version`, `.tool-versions`). When you commit to mise.toml as source of truth:

- **mise.toml is the authority.** It overrides or replaces other version files.
- **Legacy files may coexist** for tool compatibility (e.g., `.ruby-version` for rbenv/ruby-build, which mise also reads). If kept, they must match the version in mise.toml — inconsistency is a bug.
- **Do not add both.** If mise.toml declares `ruby = "3.4.1"`, you don't need `.ruby-version` unless another tool (like `rbenv`) requires it independently. Prefer removing the legacy file if nothing else needs it.

### Activation in development

Developers run:

```bash
eval "$(mise activate bash)"   # or zsh, fish
```

Or add it to their shell profile. After activation, every `cd` into the project directory automatically sets the correct tool versions on PATH.

### CI environments

CI often does not use mise — it has its own install mechanisms (apt-get, curl, setup-* actions). That's fine. The mise.toml still serves as the canonical record of which tool versions the project requires, even if CI installs them by other means.

The key rule: **when a tool version changes, update mise.toml first, then update the CI install step to match.** mise.toml is the reference; CI steps follow it. This prevents the versions from silently drifting apart.

If you do want to use mise in CI, you can:

```yaml
- run: eval "$(mise activate bash)" && mise install
- run: mise exec -- shellcheck script.sh
```

But this is optional — many CI environments are simpler with their native install paths, and mise.toml's job is to document the versions, not necessarily to provision them in every context.

### Agent environments

Containerized agent environments (harness, pi with pi-mise) handle activation automatically:

1. **pi-mise** detects `mise.toml` in the project root, runs `mise trust`, and prepends `eval "$(mise activate bash)"` to every shell invocation. The agent always runs with mise-managed tools on PATH.
2. **harness** persists `~/.local/share/mise` and `~/.local/state/mise` across runs, so installed tools survive between sessions.

### The mise trust step

When cloning a fresh repo, mise will refuse to use an untrusted `mise.toml` (security measure against malicious config). Trust it once:

```bash
mise trust
```

This persists until the file content changes. In CI and agent environments, trust should be automated as part of setup.

### Tasks (optional)

mise.toml can also define `[tasks]` — project-specific scripts that run with the correct tool versions on PATH:

```toml
[tasks]
lint = "shellcheck *.sh"
test = "bundle exec rspec"
```

Run with `mise run lint`. Tasks are optional but useful for short commands that would otherwise need a Makefile or package.json script.

## The AGENTS.md section

Add this to your agent instructions so coding agents know to use mise:

```markdown title="AGENTS.md"
## Tool Versions

This project uses `mise.toml` as the single source of truth for all tool and language versions. Do not install tools globally or via ad-hoc commands — use mise instead.

- Activate mise before running commands: `eval "$(mise activate bash)"`
- Install all declared tools: `mise install`
- Run one-off commands with correct versions: `mise exec -- <command>`
- Run defined tasks: `mise run <task>`

If you change a tool version, update `mise.toml` and run `mise install` to apply.
```

## Anti-patterns to avoid

- **Updating a tool version in CI but not in mise.toml.** mise.toml is the reference — update it first, then propagate to CI. If the two drift apart, mise.toml has lost its purpose.
- **Adding bundler/npm/pip to `[tools]`.** Package managers that ship with their language runtime (bundler with Ruby, npm with Node) are not separate mise tools. mise manages the runtime; the runtime manages its own package manager.
- **Using version ranges.** `ruby = "3"` is not the same as `ruby = "3.4.1"`. The former resolves to whatever 3.x is latest at install time, breaking reproducibility.
- **Forgetting `mise trust`.** On a fresh clone, mise will silently skip the config. Always trust as part of setup.
- **Splitting config across files.** Don't use both `mise.toml` and `.tool-versions` in the same project. Pick one (prefer `mise.toml` for richer syntax) and delete the other.

## Reference: mise.toml files across repos

### capotej/abbey

```toml
[tools]
ruby = "3.4.1"
bun = "1.2.4"
```

A Rails blog. Declares the two language runtimes it needs. Coexists with `.ruby-version` (3.4.1, matching) for rbenv compatibility.

### capotej/pi-mise

```toml
[tools]
jj = "latest"
"github:rhysd/actionlint" = "v1.7.7"
```

A pi extension. Uses `latest` for jj (acceptable trade-off for a frequently-updated VCS tool). Declares actionlint via GitHub source.

### boldblackai/harness

```toml
[tools]
jj = "0.41.0"
"github:koalaman/shellcheck" = "v0.11.0"
"github:hadolint/hadolint" = "v2.14.0"
"github:rhysd/actionlint" = "v1.7.7"
```

A containerized agent runner. Declares linters and VCS tool — all pinned to exact versions, all GitHub-sourced except jj.
