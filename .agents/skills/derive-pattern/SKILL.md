---
name: derive-pattern
description: Derives a reusable pattern file (patterns/PNNN-*.md) from source material — an AGENTS.md blurb, a config/convention lifted from another repo, or a described workflow. Computes the next pattern ID, writes the file in this repo's required format, updates the Pattern Index in README.md, and anonymizes when the source is private. Use when the user asks to "derive a pattern", add a new pattern, or turn a convention/workflow/blurb into a pattern.
---

# Derive a Pattern

This skill turns source material (a blurb, a snippet, a convention from another
repo, or a described workflow) into a properly-formatted pattern file under
`patterns/` in **this repository**, and keeps the Pattern Index in `README.md`
in sync.

## The format lives in this repo

Before writing anything, read the authoritative spec:

- `README.md` → **"Pattern Format"** (location/filename, frontmatter, structure)
- One or two existing patterns to calibrate depth — read the latest
  `patterns/P*.md` and a rich one (e.g. `P013-code-notes.md`).

Do not invent the format. Mirror what the repo already does.

## Workflow

### 1. Gather the source

Get the source material from the user: a blurb from an `AGENTS.md`, a config
snippet, a pointer at another repo/file, or a plain description of a workflow.
If the material is underspecified, ask before proceeding.

### 2. Determine provenance — the privacy gate

Establish whether the source comes from a **public** or **private** repository
(or is original to this repo). If it isn't clear, **ask the user**. This
decision gates the whole derivation:

- **Public source** → safe to cite the repo in `used_in` and include a
  `## Reference` section (repo path + URL).
- **Private source** → **anonymize** every example and all prose (see
  [Privacy rules](#privacy-rules)) and leave `used_in: []`. Never put a private
  repo in the metadata.
- **Original to this repo** → `used_in: []` unless the user names a repo.

### 3. Compute the next pattern ID

```bash
ls patterns/P*.md            # find the highest numeric prefix
```

Take the highest `PNNN`, increment, zero-pad to **3 digits** (`P013` → `P014`).

### 4. Derive name + filename

- `name` is **kebab-case** (`rfc-process`, `using-mise`, `code-notes`).
- Filename: `patterns/PNNN-<name>.md`.

### 5. Write the pattern file

Use the [standard structure](#standard-pattern-structure) below. Fill the
frontmatter:

- `id` — the `PNNN` from step 3
- `name` — from step 4
- `date` — today's date, ISO `YYYY-MM-DD`
- `author` — the configured VCS identity (`git config user.name` / `jj config
  user.name`)
- `used_in` — per the [privacy gate](#2-determine-provenance--the-privacy-gate)

### 6. Update the Pattern Index in README.md

Add a row to the **Pattern Index** table in `README.md`, in **numeric order**:

```markdown
| [PNNN](patterns/PNNN-name.md) | Human Title | One-line description. |
```

(Per `AGENTS.md`: every new pattern under `patterns/` must get a row here, kept
in numeric order.)

### 7. Confirm `used_in`

If `used_in` is empty and the source is public, ask the user which public repo
the pattern is used in. Never invent a value; an honest `[]` beats a fabricated
reference.

## Privacy rules

When the source is a **private repository** (or contains anything not safe to
publish):

1. **Genericize all examples.** Replace real names, identifiers, org/usernames,
   internal paths, hostnames, secrets, and project-specific terms with generic
   stand-ins (`example-repo`, `my-org`, `UserService`, `https://example.com/…`).
   Do this in codeblocks **and** prose.
2. **Leave `used_in: []`.** Do **not** put the private repo path in the
   frontmatter metadata.
3. **No `## Reference` to the private source.** Either omit the section or, if
   the convention is genuinely portable (e.g. ported from a public style
   guide), cite only the public origin — never the private repo.

When the source is **public**, you may cite it in `used_in` and add a
`## Reference` section with the repo and a URL.

## Standard pattern structure

Mirror the depth of existing patterns. A fenced block representing a **whole
file** declares its language **and** filename (`title="..."`); illustrative
fragments and shell commands use the language only. This matches how the
existing patterns are written — whole files like `mise.toml` carry a title,
but one-off commands (`uv run ruff check .`) and config fragments use the
language alone.

````markdown title="patterns/PNNN-name.md"
---
id: PNNN
name: name
date: YYYY-MM-DD
author: Author Name
used_in: ['github.com/org/example-repo']
---

# Human-Readable Title

One-or-two paragraph intro: what the pattern is and why it exists.
(If public, note the provenance here.)

## When to use

- Bullet list of situations/projects this fits.

## How it works

Prose explaining the mechanics, broken into subsections. Each subsection
ships a fenced codeblock (whole-file blocks carry `title="..."`; fragments/commands use the language only):

```language title="path/to/file.ext"
<the actual config / snippet / template>
```

### Sub-mechanic
...etc.

## The AGENTS.md section

The copy-pasteable block an adopter drops into their own `AGENTS.md`:

```markdown title="AGENTS.md"
## Some Convention

- The bullet the agent follows.
```

## (Optional) A template / skeleton

```markdown title="path/to/template"
<the skeleton a new instance starts from>
```

## Anti-patterns to avoid

- **The failure mode.** Why it's wrong.

## Reference
<!-- ONLY for public sources -->
### github.com/org/example-repo

Where it came from, with a URL.
````

Not every pattern needs every section — but "When to use", "How it works", at
least one `AGENTS.md`-ready codeblock, and (for anything non-obvious)
"Anti-patterns to avoid" should be the baseline.

## Done checklist

- [ ] File at `patterns/PNNN-<name>.md` with correct frontmatter.
- [ ] Whole-file fenced blocks carry a language **and** filename (`title="..."`); fragments/commands use the language only (matches existing patterns).
- [ ] Pattern Index row added to `README.md`, in numeric order.
- [ ] Privacy handled: if the source is private, examples are anonymized and
      `used_in` is `[]` with no private `## Reference`.
- [ ] `used_in` confirmed with the user (or deliberately left `[]`).
