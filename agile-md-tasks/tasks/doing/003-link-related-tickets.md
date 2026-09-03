---
id: "003"
title: "link related tickets"
ticket: "development"
type: "feat"
parent: ""
branch: "feature/shell-completions"
related: [002]
created: "2026-08-04"
tags: []
---

## Notes

Tickets that depend on other tickets should be able to link to them, so a
RELATED list is needed on every ticket, empty by default.

Split out of 002, which originally carried this alongside the admin ticket type.

## Acceptance criteria

- [ ] Every new ticket has a RELATED list, empty by default
- [ ] A ticket can be created with links to tickets that already exist
- [ ] Two existing tickets can be linked together after the fact
- [ ] Both ends of a link are recorded, so it reads the same from either ticket
- [ ] A link to a ticket that doesn't exist is rejected when it is made
- [ ] Links survive a ticket being renamed or moved between columns
