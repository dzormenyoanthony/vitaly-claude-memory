# Vitaly — Claude Code Memory

Persistent, file-based memory for Claude Code sessions on the
[Vitaly](https://github.com/dzormenyoanthony) Flutter project. Each file
holds a single durable fact — project state, decisions, gotchas, and
working-style feedback — that would otherwise be lost between sessions.

This repo is the backing store for
`~/.claude/projects/C--Users-hp-AndroidStudioProjects-Vitality/memory/`.

## Layout

- **`MEMORY.md`** — the index. One line per memory (`- [Title](file.md) — hook`),
  loaded into context at the start of every session. No memory content
  lives here, only pointers.
- **`<slug>.md`** — one fact per file, with YAML frontmatter:

  ```markdown
  ---
  name: <short-kebab-case-slug>
  description: <one-line summary, used to judge relevance during recall>
  metadata:
    type: user | feedback | project | reference
  ---

  <the fact; feedback/project entries add **Why:** and **How to apply:** lines>
  ```

  Files cross-link with `[[other-slug]]`.

### `type` values

| type        | contents                                                            |
|-------------|--------------------------------------------------------------------|
| `user`      | who the user is — role, expertise, preferences                    |
| `feedback`  | how Claude should work here — corrections and confirmed approaches |
| `project`   | ongoing work, goals, constraints not derivable from code or git    |
| `reference` | pointers to external resources (URLs, dashboards, tickets)         |

## Conventions

- One fact per file; update the existing file rather than adding a duplicate.
- Delete a memory once it's proven wrong.
- Don't record what the codebase already captures (structure, past fixes,
  git history, `CLAUDE.md`).
- Relative dates are converted to absolute before saving.
- Line endings are pinned to LF via `.gitattributes`.

## Not in this repo

No secrets, credentials, keystore passwords, tokens, or real user health
data. Signing-certificate SHA-1/SHA-256 fingerprints are public
identifiers and may appear.
