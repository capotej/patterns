# Patterns

This repo houses a collection of common patterns that can be mixed and matched together.

## Pattern Format

### Location & Filename

Patterns are markdown files that live inside of `patterns/`, named sequentially like: `patterns/P001-name-of-the-pattern.md`.

### Frontmatter Metadata

Patterns must contain the following YAML frontmatter at the beginning of their file:

```yaml
---
id: P001
name: name-of-the-pattern
date: 2026-07-15
author: John Doe
used_in: ['github.com/capotej/example-repo']
---
```

### Structure

Patterns contain fenced markdown codeblocks (always containing language and filename) surrounded by explanations of how/when they should be used.

Example:

```typescript title="userService.ts"
export class UserService {
}
```

## Usage

Point your agent at this repository with a prompt like "Implement patterns P031 and P021 in this project".


