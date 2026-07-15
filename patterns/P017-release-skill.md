---
id: P017
name: release-skill
date: 2026-07-15
author: Julio Capote
used_in: ['github.com/boldblackai/harness']
---

# The Release Skill (Agent-Driven Release Runbook)

Cutting a release is a long, fiddly, order-sensitive sequence — pre-flight checks, version bump, changelog, build, publish, tag, GitHub release, verify CI — where every step has a failure mode (publishing the wrong version, tagging before npm succeeds, declaring victory while CI is red). Done ad hoc, it depends on whoever is at the keyboard remembering the whole runbook and not skipping the boring gates.

The Release Skill turns that runbook into a **discrete agent skill**: a `SKILL.md` whose frontmatter trigger phrases make the agent load it whenever release work is requested, and whose body is a numbered pipeline the agent executes deterministically. The skill is the single source of truth for "how we release here" — the agent never improvises release steps, and a human can read the file to see exactly what will happen.

Ported from the `release` skill in `boldblackai/harness`, which releases an npm package using jj for version control and drives Docker-image variants off the GitHub release webhook.

## When to use

- Any project with a repeatable release process (npm package, CLI, container image) that an agent should be able to drive end to end.
- Any repo where releases have bitten you because a step was skipped or done out of order — the skill makes the gates non-optional.
- jj-based repos especially: the jj workflow for describe/push/tag differs enough from git's muscle memory that a codified runbook is worth more than `git release` folklore.
- Projects that ship artifacts with external dependencies (Docker base images, vendored upstream agents) whose own changelogs belong in the release notes.

## How it works

### The skill is a runbook, loaded by trigger phrases

The skill lives at `.agents/skills/release/SKILL.md`. Its frontmatter `description` is both documentation and the **trigger contract** — the phrases that make the agent load the skill instead of winging it:

```markdown title=".agents/skills/release/SKILL.md"
---
name: release
description: Automate releasing the package. Use this skill whenever the user
  wants to cut a release, publish a new version, bump the version, tag a
  release, update the CHANGELOG, or run npm publish. Triggers on phrases like
  "release version X", "cut a release", "publish", "bump to X.X.X", "tag this
  release", "release the project", or any combination of version bumping +
  publishing intent. Always use this skill for release work — don't attempt
  ad-hoc release steps without it.
---
```

The last sentence is load-bearing: it tells the agent that release commands are **not** to be handled with a quick `npm publish` on its own — they must go through the skill. Without it, an agent will happily publish without bumping the version or writing a changelog.

### The pipeline is numbered, and each step ships its own commands

The body is a sequence of numbered steps. Numbering does two things: it gives the agent a resumable cursor (it knows where it is), and it forces the author to commit to an order. The shape, in full:

```text
Step 1:  Pre-flight checks          (abort on failure)
Step 2:  Determine the new version  (explicit, or inferred from commits)
Step 3:  Collect commits since last tag
Step 4:  Update CHANGELOG.md
Step 5:  Bump version in package.json
Step 6:  Build
Step 7:  Commit and push the release
Step 8:  Publish to npm             (manual OTP gate)
Step 9:  Create and push the tag
Step 10: Create the GitHub release
Step 11: Verify CI succeeded        (don't report success until green)
```

Each step is a short prose block followed by the exact commands. An agent reads top to bottom and never has to decide what comes next.

### Stop-the-line pre-flight gates (Step 1)

The first thing the skill does is **abort** if the preconditions aren't met, rather than discovering the problem mid-release. Three gates:

- **`main` is in sync with the remote.** In jj, remote bookmarks are `main@origin` (not `origin/main`). Compare commit IDs:

```bash
jj log -r "main" --no-graph -T 'commit_id ++ "\n"'
jj log -r "main@origin" --no-graph -T 'commit_id ++ "\n"'
```

If they differ, abort: *"local main is ahead of main@origin — push first."*

- **Clean working state.** `jj status` — warn if there are changes beyond the `package.json` + `CHANGELOG.md` the release is about to create.
- **README is current.** Read the commits since the last tag (Step 3) and check whether any introduces user-visible behavior (new CLI flags, options, agents) that `README.md` doesn't reflect. If so, abort and ask for the doc update first.

The point of these gates is that a half-broken release is much more expensive to clean up than a delayed one. Aborting early is the whole feature.

### Version inference from commits (Step 2)

If the user didn't name a version, infer a semantic bump from the commits since the last tag:

- **patch** (default) — bug fixes, docs, tooling, and *also* new features (`feat:`)
- **minor** — only on request, or commits adding new user-facing CLI flags/options/agents
- **major** — only on request, or explicit breaking-change messages

Defaulting to **patch even for `feat:`** is a deliberate call: in this project, features don't automatically escalate the version; a minor bump is a conscious human decision. State the chosen version and the reasoning to the user before proceeding.

### CHANGELOG generation (Step 4)

The changelog entry is the heart of the release. It is **not** a raw commit dump — it leads with 1–3 sentences of prose summarizing what actually changed (new features, fixes, notable improvements), with a concrete inline example for any new user-visible feature. Then the raw commit list goes underneath. If `CHANGELOG.md` already exists, the new entry is inserted immediately after the `# Changelog` header, before older entries.

```bash
LAST_TAG=$(git describe --tags --abbrev=0 2>/dev/null)
git log ${LAST_TAG}..HEAD --oneline     # → "- <short-hash> <message>" bullets
date +%Y-%m-%d                           # today's date for the header
```

```markdown title="CHANGELOG.md"
# Changelog

## [1.8.1] - 2026-07-15

### Summary
Adds a `--watch` flag that re-runs on file change, and fixes a crash when the
config file is empty. Try it with `harness --watch src`.

### Changes
- a1b2c3d feat: add --watch flag
- d4e5f6a fix: handle empty config without crashing
- 789abcd docs: clarify install steps
```

#### Dependency and upstream-note sections (optional)

For projects that bundle upstream artifacts, two optional subsections make the changelog genuinely useful:

- **`### Dependency Updates`** — diff the Dockerfiles (or lockfiles) against the last tag; for every tool version that moved, emit `- updated <package> from <old> to <new>`.

```bash
git diff ${LAST_TAG}..HEAD -- Dockerfile Dockerfile.* | grep -E '^\+|^-' | grep -iE 'pin|version|@'
```

- **`### Upstream Release Notes`** — when a pinned upstream (e.g. the agent runtime, a vendored CLI) was bumped, fetch the release notes for **every** version between the old pin (exclusive) and the new pin (inclusive) and condense each to 2–4 bullets. Run the fetches in parallel:

```bash
# enumerate intermediate npm versions, then gh-release the notes for each
npm show @scope/agent versions --json
gh release view v0.70.2 --repo owner/agent --json tagName,body
```

Condense, don't paste verbatim — the changelog summarizes; the upstream release page is the canonical source. Omit both subsections entirely when nothing relevant changed.

### VCS-aware commands (Steps 5, 7, 9) — and why not `npm version`

Edit the `version` field in `package.json` **directly**. Do **not** use `npm version`: it creates its own git commit, which fights the jj workflow (jj snapshots working-copy changes into the current commit; an extra git commit lands in the wrong place and corrupts the bookmark model).

In jj, the release commit is the working-copy commit — describe it, advance the bookmark, push, then tag:

```bash
# Step 7 — commit & push
jj describe -m "release v<version>"
jj new
jj bookmark set main -r @-
jj git push --bookmark main

# Step 9 — tag pointing at the release commit (the one behind the empty @)
jj tag set v<version> -r @-
git push --tags          # jj can't push tags; use git
```

`@` is jj's current (empty, after `jj new`) working copy; `@-` is the release commit just behind it. The tag must point at the commit that was actually pushed.

### Manual gates where automation can't reach (Step 8)

`npm publish` needs an OTP, so it can't be automated. The skill treats this as an explicit **wait gate**: tell the user to run it, and **stop** until they confirm success:

> *"Please run `npm publish` (with `--otp=<code>` if prompted). Let me know when it succeeds and I'll continue."*

The tag (Step 9) is created **after** publish succeeds, not before — so a failed publish never leaves an orphan tag pointing at an unpublished version.

### Post-flight: don't report success until CI is green (Step 11)

Creating the GitHub release triggers the release workflow (which builds and pushes the versioned container images, in this project). The release isn't done until that workflow is green. Poll it:

```bash
gh run list --repo <owner>/<repo> --event release --limit 1
gh run view <run-id> --repo <owner>/<repo>
```

Watch the jobs that actually publish artifacts — the image-build variants and the `merge-variant` jobs that push versioned tags are the most failure-prone. If any job failed, rerun the failed ones and re-verify:

```bash
gh run rerun <run-id> --failed --repo <owner>/<repo>
```

Only report the release as complete once the entire workflow is green. A release that's "published but CI is red" is not a finished release.

## The AGENTS.md section

```markdown title="AGENTS.md"
## Releases

Releases are driven by the `release` skill (`.agents/skills/release/SKILL.md`),
not done ad hoc.

- Load the `release` skill whenever release work is requested ("cut a release",
  "publish", "bump to X", "tag this release"). Do not run release steps
  (`npm publish`, version bumps, tags) outside the skill.
- Pre-flight aborts the release if `main` isn't in sync with the remote, the
  working tree is dirty, or `README.md` is out of date vs. the commits being
  released.
- Infer the version from commits when one isn't given; default to **patch**
  (features included). State the chosen version and why before proceeding.
- Write the CHANGELOG: 1–3 sentences of prose summary + a concrete example for
  any new user-visible feature, then the raw commit bullets. Add
  `### Dependency Updates` / `### Upstream Release Notes` sections only when a
  Dockerfile/lockfile or pinned upstream actually moved.
- Bump `version` in `package.json` by hand — never `npm version` (it commits
  and fights jj). Publish is a manual OTP gate; create the tag only after
  publish succeeds.
- The release is not complete until the release-triggered CI workflow is fully
  green. Rerun failed jobs and re-verify before reporting success.
```

## Skill skeleton

A new project can start from this skeleton and fill in its own version-control, package manager, and artifact specifics:

```markdown title=".agents/skills/release/SKILL.md"
---
name: release
description: Automate releasing <package>. Use this skill whenever the user
  wants to cut a release, publish a new version, bump the version, tag a
  release, update the CHANGELOG, or publish. Triggers on "release version X",
  "cut a release", "publish", "bump to X.X.X", "tag this release". Always use
  this skill for release work — don't attempt ad-hoc release steps without it.
---

# Release Skill for `<package>`

## Step 1: Pre-flight (abort on failure)
- main bookmark in sync with remote
- clean working state
- README current vs. commits being released

## Step 2: Determine the new version
- explicit, or inferred from commits (default patch)

## Step 3: Collect commits since last tag

## Step 4: Update CHANGELOG.md
- prose summary + example for new features, then commit bullets
- (optional) Dependency Updates / Upstream Release Notes

## Step 5: Bump version in package.json (by hand, not `npm version`)

## Step 6: Build

## Step 7: Commit and push the release

## Step 8: Publish (manual OTP gate — wait for confirmation)

## Step 9: Create and push the tag (after publish succeeds)

## Step 10: Create the GitHub release

## Step 11: Verify CI is green before reporting success
```

## Anti-patterns to avoid

- **Ad-hoc release steps outside the skill.** `npm publish` on its own, or a hand-rolled version bump, skips the gates and the changelog. The trigger contract exists to prevent exactly this — every release command routes through the skill.
- **Using `npm version`.** It creates a git commit, which lands in the wrong jj commit and corrupts the bookmark model. Edit `package.json` by hand.
- **Tagging before publish succeeds.** A failed `npm publish` then leaves an orphan `vX.Y.Z` tag pointing at an unpublished version. Create the tag only after publish is confirmed.
- **Skipping the README-up-to-date gate.** Releasing user-visible changes with a stale README ships undocumented behavior. The gate makes "update the docs" non-optional.
- **Dumping raw commits as the changelog.** A bullet list with no prose summary forces every reader to reverse-engineer what the release *means*. Lead with the human summary; the commits are the appendix.
- **Pasting upstream release notes verbatim.** The changelog summarizes (2–4 bullets per upstream version). The upstream release page is already the canonical full text — duplicating it bloats the changelog and guarantees drift.
- **Reporting success while CI is red.** "Published" ≠ "released." The release-triggered workflow builds and ships the artifacts; if it's failing, the release is incomplete. Rerun and re-verify.
- **Reporting the version as a minor/major bump just because a `feat:` commit exists.** In this project features default to patch. A minor or major bump is a conscious decision, not an automatic consequence of a commit prefix.
- **Inventing an intermediate-version list.** When fetching upstream notes, enumerate the actual versions between old and new pins (e.g. via `npm show … versions --json`) — don't guess. Missing an intermediate version drops its notes silently.

## Reference

### github.com/boldblackai/harness

The `release` skill at `.agents/skills/release/SKILL.md` is the canonical instance. It releases the harness npm package via jj, generates the changelog with `### Dependency Updates` (Dockerfile tool pins) and `### Upstream Release Notes` (fetched from the pi-coding-agent, opencode, and hermes-agent release pages), then drives the GitHub release that triggers the multi-variant container build workflow.

`https://github.com/boldblackai/harness`
