---
id: P013
name: code-notes
date: 2026-07-07
author: Julio Capote
used_in: ['gitlab.haskell.org/ghc/ghc']
---

# Code Notes (GHC-style "Notes")

One of the hardest problems in a large, long-lived codebase is keeping technical documentation up to date. There is no silver bullet, but the GHC project's low-tech mechanism has held up over twenty years: **Notes**.

A Note is a block comment, set off by itself, with a heading in standard form: `Note [Title]`. At each call site that depends on it, a short comment — `See Note [Title]` — points at it by exact title. The title is the Note's identity: one `grep` finds the definition and every reference. Because Notes are named, a single Note documents a concept once and is pointed at from every site that relies on it.

Ported from the GHC coding style, section "Using Notes".

## Why (the two bad choices)

The problem Notes solve: a careful programmer hits an invariant — "this data type has an important property" — and faces two unsatisfactory choices:

1. **Add it inline.** The comment bloats the declaration so much that the constructors become hard to see.
2. **Document it elsewhere.** The invariant drifts from the code and goes out of date. Over twenty years, everything goes out of date.

A Note is the third option: the explanation lives *out of the way* (so the code stays readable) but is *referenced by an exact title from the site* (so it's trivially found and stays connected). The same precision also replaces the old, fragile habit of `// see the comment above` — which, after a few years, often referred to a comment that was no longer above, or gone altogether.

## When to use

- Any codebase where non-obvious invariants, subtle "why" decisions, or fragile interactions need to be recorded near the code that depends on them.
- Agent-driven repos especially: Notes give an agent a durable, greppable place to *find* the reasoning behind tricky code (instead of re-deriving it) and a single, canonical place to *record* it when it discovers something non-obvious.
- Any project that wants "why" comments to be authoritative rather than scattered, stale folklore.

## How it works

### The standard heading form

A Note is a block comment whose first line is `Note [Title]`, followed by a line of tildes matching the title width. The title (in square brackets) is the Note's identity — every reference uses it verbatim, so a single search finds the definition and every site that depends on it. The `~~~` underline is part of the standard form, not decoration.

```haskell title="src/Type.hs"
{-
Note [Equality-constrained types]
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
The type   forall ab. (a ~ [b]) => blah
is encoded like this:

   ForAllTy (a:*) $ ForAllTy (b:*) $
   FunTy (TyConApp (~) [a, [b]]) $
   blah
-}
```

In languages without block comments, the same convention carries over using line comments — the `Note [Title]` header and `See Note [Title]` reference are what matter:

```python title="src/parser.py"
# Note [Why we two-pass the input]
# ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
# We scan once to build the symbol table and again to resolve refs,
# because a single pass can't forward-reference. See Note [Symbol
# resolution order] for why the table is keyed on the mangled name.
```

### Reference by exact title — not "the comment above"

At the point where the Note is relevant, add a short comment pointing at it:

```haskell title="src/Type.hs"
data Type
  = FunTy Type Type   -- See Note [Equality-constrained types]
  | ...
```

Use the title **verbatim**. The precision of `See Note [Equality-constrained types]` is vastly better than `see the comment above`: it names exactly one comment, and it survives re-ordering, deletion, and years of edits — none of which "the comment above" survives.

### Bidirectional, and one Note can have many references

You can go from the code *to* the Note (via `See Note [..]`) and from the Note *back* to the code (the Note names the functions/types it's about). The same Note is often referenced from multiple points. Prefer pointing at an existing Note over re-explaining — if the same explanation is appearing twice, it probably wants to be one Note.

### Overview Notes

Some questions — "how is `unsafeCoerce` implemented?", "what does it mean for two types to be equal here?" — have no single bit of code that answers them. There can, however, be one Note. An **overview Note** summarises how the codebase deals with a topic, and should cover:

- **Topic** — identify the problem.
- **Design** — the big picture.
- **Moving parts** — the key bits of code that implement the feature, referred to by name (and those functions/types should, in turn, reference the overview Note back).
- **Wrinkles** — the non-obvious corners needing special treatment, with pointers to the specific code that handles each.

Give each wrinkle a name like `(W7)` so you can reference it elsewhere: `See wrinkle (W7) of Note [Wombat]`. Precision compounds. Many merge requests are dramatically easier to review if they open with a suitable overview Note.

### Be specific; use concrete examples

Don't just say "we reject overlapping instances in situation X" — point directly to the code that implements that decision. Offer concrete examples (a reproducing input, a ticket, a testsuite case). A general principle is often best illuminated by seeing it play out in one particular example.

It is easy for these code references to go out of date. **That's accepted as the lesser evil** — a concrete pointer that goes stale is more useful than a vague but immortal one. When you touch the code, check the Note's pointers too.

### Notes describe the present state, not the journey

A commit message describes a *change*; a comment or Note describes a *state*. The Note should always describe how things are *now*, not the journey that got there. At the moment you write it, the journey is vivid; in ten years it's a distraction, and at worst it makes the Note incomprehensible without knowledge of the previous state.

Two sanctioned nuances:

- **Design choices.** A Note is a good place to record the *reasoning* behind a decision — present-state rationale ("we chose X because Y; changing it would break Z"), not travelogue. Without it, someone may later reverse the decision without realising the consequences.
- **Historical notes.** Sometimes a previous state of affairs is genuinely worth recording. Put it in a clearly signposted section — `Historical note: ...` — that signals a reader doesn't need it to understand the current code. The fence is what makes it safe.

When you change the code, *rewrite* the Note for the new state — don't *append* a chronicle.

### Enforce with a lint check

The convention's whole premise is that `See Note [Title]` resolves to a real Note. GHC runs a lint check in CI that flags references to Notes that don't exist. The portable version is the same idea: any check that greps every `See Note [X]` and confirms a `Note [X]` definition exists (and ideally flags a `Note [X]` with no remaining references). A linter that only runs locally is advisory; the gate is what makes broken references fail.

## The AGENTS.md section

```markdown title="AGENTS.md"
## Code Notes

Non-obvious invariants, subtle "why" decisions, and fragile interactions are
documented as **Notes**: titled block comments referenced by exact title.

- Define a Note as a block comment with a `Note [Title]` header (and `~~~`
  underline). Keep the title unique and stable — it's the Note's identity.
- Reference it from any site with `See Note [Title]`, verbatim. Never write
  "see the comment above"; name the Note. One `grep` should find the definition
  and every reference.
- One Note per concept; reference it from every site that depends on it rather
  than re-explaining.
- For anything spanning multiple files or a whole feature, write an **overview
  Note**: Topic, Design, Moving parts (named code), Wrinkles (named, e.g. (W7)).
- Be concrete: point to the implementing code, a repro, a ticket. Pointers can
  go stale — that's the lesser evil.
- **Describe the present state, not the journey.** A commit describes a change;
  a Note describes a state. Record design *rationale* (present-state); if old
  state truly matters, fence it under `Historical note:`. When you change code,
  rewrite the Note — don't append a changelog.
- References to Notes must resolve; CI should flag a `See Note [X]` with no
  matching `Note [X]`.
```

## Anti-patterns to avoid

- **Narrating the journey.** "We used to do X, then Y broke, so now Z" — the cardinal sin this pattern exists to prevent. History lives in commits (`git blame`); the Note is the current description.
- **Appending instead of rewriting.** Editing code and tacking the change onto an existing Note turns it into a changelog. Rewrite it for the new present state.
- **"See the comment above."** The whole point of the title is precision. "Above" breaks under re-ordering, deletion, and time — use `See Note [Title]`.
- **Untitled, anonymous comments.** A `// TODO: tricky` or a bare paragraph can't be referenced or searched. If it's worth saying, give it a `Note [Title]` it can be pointed at.
- **Paraphrased or typo'd references.** `See Note [Two pass input]` when the Note is `[Why we two-pass the input]` greps to nothing and strands the reader. The title is an exact link.
- **Renaming a Note without updating references.** The title is referenced verbatim elsewhere. Grep for the old title before renaming — or don't rename.
- **No overview Note on a multi-file feature.** Reviewers (and agents) arriving cold have no entry point. A Topic/Design/Moving-parts/Wrinkles overview makes the change legible.
- **Deleting a Note and leaving dangling references.** When removing a Note, grep for its title and clear the `See Note [..]` pointers.
- **Vague Notes.** "We reject this case" with no pointer to the implementing code. Be specific; concrete-but-stale beats vague-but-immortal.
- **No lint gate.** If `See Note [X]` can silently dangle, the convention rots. The CI check is what makes the title a real link.

## Reference

### gitlab.haskell.org/ghc/ghc

The canonical source of the convention. GHC's own codebase has thousands of Notes and grows daily; `grep -rn "Note \[" compiler/` returns definitions and references across the tree. The convention is documented in the GHC wiki's coding style guide, section 2 "Using Notes":

`https://gitlab.haskell.org/ghc/ghc/-/wikis/commentary/coding-style#2-using-notes`

Subsections: 2.1 the basic idea, 2.2 Overview Notes, 2.3 Being specific (using examples), 2.4 Notes describe the present state, not the journey. GHC additionally runs a CI lint check that identifies references to Notes that don't exist.
