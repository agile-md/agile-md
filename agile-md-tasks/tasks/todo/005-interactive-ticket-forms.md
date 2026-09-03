---
id: "005"
title: "interactive ticket forms"
ticket: "development"
type: "feat"
parent: ""
branch: "feature/interactive-ticket-forms"
related: []
created: "2026-08-04"
tags: []
---

## Notes

Flags are only ergonomic when you remember them. Every argument should be
optional in a terminal, and creating a ticket should carry on into the body
rather than stopping at the metadata.

Delivered alongside [[002-add-admin-ticket-type]], which settled what the form
actually asks for.

## Acceptance criteria

- [x] `amd new` with no arguments asks for what it needs
- [x] `amd start`/`done`/`back`/`show`/`edit` offer a task picker with no ref
- [x] A template that uses `extra.<name>` is asked for it
- [x] The form finishes in $EDITOR with the rendered ticket
- [x] Nothing is written if the editor exits non-zero or empties the file
- [x] Nothing prompts unless stdin and stdout are a terminal
- [x] Titles are validated against git ref rules as they are typed
