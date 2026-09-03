# agile-md — session transcript

Raw log: `transcript.jsonl`. Rendered 2026-08-21.

## You

the problem is you only get the gui if you pass the gui argument. can we have cargo create two binaries amd and amdapi or amdui and amdapi

## Claude

Before I wire this up — the split itself is easy (a second `[[bin]]`), but the naming decides how much breaks. `tests/test.sh`, `install.sh`, `release.yml` (`bin-name: amd`) and the README all assume the binary on PATH is `amd`.

## Claude

Now the changes.

## Claude

Now install.sh, so both land on PATH.

## Claude

Now the `amd gui` help text, the README, and a test.

## Claude

Done — `cargo build` now produces both binaries.

**What changed**

- [Cargo.toml:21-31](Cargo.toml#L21-L31) — a second `[[bin]]`, `amdui`, with `required-features = ["gui"]` so `--no-default-features` simply doesn't produce it (verified: that build still succeeds and emits only `amd`).
- [src/bin/amdui.rs](src/bin/amdui.rs) — ~50 lines over `agile_md::gui::run()`. No clap: it answers `-h`/`-V`, rejects anything else with the usage, and otherwise opens the window. Errors print as `amdui: {err:#}`, matching `amd`.
- `amd gui` is untouched and still works — it's the same call, and it's the only way in for anyone on a prebuilt install (see below).
- [install.sh](install.sh) — a source build now installs `amdui` alongside `amd`; both copy paths guard with `if [ -f ... ]`, so a `--no-default-features` build or an `amdui`-less zip installs cleanly.
- [tests/test.sh:301-317](tests/test.sh#L301-L317) — three assertions on the new binary (version, help, bad-arg), skipped with a notice when there's no `amdui`. All three pass.
- [README.md](README.md) and [CLAUDE.md](CLAUDE.md) updated: two binaries, `amdui` as the way into the board.

**Two things you should know**

The release zip won't carry `amdui`. `mrdoodles/rust-release` packages exactly one binary (`bin-name` — I read [the workflow's Package step](/Users/mrdoodles/develop/github/mrdoodles/lite-actions/rust-release/.github/workflows/rust-release.yml)), so a tagged release still ships `amd` alone. That costs nothing functionally since `amd gui` opens the same window, but shipping `amdui` prebuilt means teaching `rust-release` to take a list of binaries. I've noted it in [release.yml](.github/workflows/release.yml) and CLAUDE.md rather than editing another repo.

`tests/test.sh` was already failing 29 assertions on this branch, before I touched anything — I confirmed it by stashing (86 pass / 29 fail before, 89 / 29 after). They're all fallout from tickets now landing in `backlog/` while the suite still asserts `tasks/todo/001-first-task.md`. Separate from this work; say the word and I'll bring the suite up to the backlog layout.

I also moved `#[cfg(test)] mod tests` to the end of [src/gui/richtext.rs](src/gui/richtext.rs) — a pre-existing `clippy::items_after_test_module` error that CI gates on. `cargo fmt --check`, `clippy -D warnings`, `cargo test` and shellcheck are all clean now.

## You

we are in a better shape  than we were but there is still a lot of confusion going on here and misalignments. we need to start afresh with a new architecture. an agilemd-lib which will provide a consistent set of methods, amd-cli which will be the command-line interface which uses the methods in the amd-lib and a amd-ui which will be the graphical interface also using the methods in the amd-lib. The rule is that the amd-lib. we also need an amd-adr repository documenting the architectual design features. This approach will ensure that we do not drift from the design. Any thoughts on this ?

## Claude

Let me check a few things before answering, so this is grounded in what the code actually does rather than what the docs say it does.

## Claude

Broadly yes — but I'd push back on two parts of it, and I want to flag that the diagnosis and the cure aren't quite matched up.

## The architecture you're describing is roughly what already exists

I checked the boundary rather than trusting the docs:

- `src/gui/` has **zero** references to `inquire`, `clap` or `richrs`.
- The GUI writes no task files — the only `fs::write` in it is test code in [settings.rs:159](src/gui/settings.rs#L159). Moves genuinely go through `Board::move_task`.
- Both front ends already sit on `agile_md::*`.

So a three-crate split won't *create* the separation — it's already there by convention. What it buys is **enforcement**: if `amd-lib/Cargo.toml` simply doesn't list `clap`, `inquire`, `egui` or `richrs`, a violation stops being a code-review catch and becomes a compile error. That is a real and sufficient reason to do it. Just don't expect it to fix the confusion, because the confusion isn't structural.

## Where the drift actually is

Every one of these is a place where two sources of truth disagree, and nothing failed when they diverged:

1. **The spec is red and has been ignored.** [tests/test.sh](tests/test.sh) asserts `tasks/todo/001-first-task.md`; `amd new` lands in `backlog/`. 29 assertions failing. CLAUDE.md calls this suite "the compatibility contract" — a contract nobody is enforcing is where drift comes from.
2. **The CLI's own help text is stale.** [main.rs:35-41](src/main.rs#L35-L41) tells users "There are two ticket types: development and admin" and names conventional-commit types (`feat`, `fix`, `docs`), while the real `--branch-type` values are `feature`/`bugfix`/`hotfix`/`release`/`chore` and CLAUDE.md says ticket types are gone.
3. **The library prints to stdout** — [board.rs:513](src/board.rs#L513) `println!("moved {} -> {to}/")`. When the GUI drags a card, that goes nowhere.
4. **`render.rs` is a terminal renderer living in the shared library** — `richrs`, `IsTerminal`, OSC-8 hyperlinks. Under your split it's `amd-cli` code, not `amd-lib`.
5. **The two front ends disagree about ordering**: the GUI writes `order`, `amd ls` sorts by id. CLAUDE.md already admits this is "a real inconsistency."

A workspace split fixes #3 and #4 by construction. It does nothing for #1, #2 and #5 — those need the suite brought back to green and kept there.

## Two disagreements

**"Start afresh" — make it a move, not a rewrite.** Those 5,200 lines encode edge cases that took real work: `deunicode` slugging so a non-Latin title still yields a valid branch, the junk drawer's `git rm --cached` dance around gitignore, SIGPIPE, the `git mv` → `fs::rename` fallback. A rewrite pays for all of that twice. The extraction is mechanical: a workspace, four `Cargo.toml`s, `render.rs` moves to the CLI crate, and the `println!`s become return values. Same code, enforced boundary.

**A separate `amd-adr` repo will drift faster than the code.** That's the exact failure mode you're trying to stop: an ADR in another repo can't be reviewed in the same PR as the diff that violates it. In `docs/adr/` the decision and the code that breaks it move in one commit, and CI can check. The one case where you're right and I'm wrong: if these ADRs are **cross-repo standards** for the whole mrdoodles family — rust-release, conventional-validator, agile-md — then a shared repo is correct. Which is it?

## Naming

Crate names and binary names are independent. I'd keep the binaries `amd` and `amdui` — installed, documented, in `install.sh` and the release URL — while the crates become `amd-lib` / `amd-cli` / `amd-ui`. Renaming what users type gains nothing.

## One thing I need

Your sentence cut off: *"The rule is that the amd-lib."* That rule is the whole point of the exercise, so I don't want to guess it. My assumption if you don't say otherwise: **amd-lib is the only thing that touches the filesystem and git; it never prompts, never prints, and never knows whether a terminal is attached — the front ends only ask and draw.** Is that it, or did you mean something stronger?

Tell me the rule and the ADR scope, and I'll draft the workspace layout plus ADR-0001 through -0004 covering the decisions already made (status-is-the-folder, git-as-audit-trail, templates-as-the-form, one-lib-two-front-ends).

## You

point 2, this is a problen the two two ticket type  is obsolete we abandoned that approach.  Point 5 ordering is difficult for the shell version particularly with the prioritories low medium and high unless it is possible to create some shell enums.. The shell version is missing commands that the gui version has. if we have a single api written in rust that both the rust gui and the shell cli use then we will bring in some consistency

## Claude

Three things, one of which changes the constraint you're working under.

## There is no shell version any more

The CLI is Rust — [src/main.rs](src/main.rs), clap, ~900 lines. The bash script was v3; v4 replaced it. So "unless it is possible to create some shell enums" isn't a constraint you have to design around: enums are already a solved problem in this codebase, and [src/value.rs](src/value.rs) exists *specifically* for it. A `TypedValueParser` advertises its possible values to `--help` and to shell completions while keeping our own validation and error text.

A `priority` of low/medium/high is roughly: 15 lines in `value.rs`, a frontmatter key, a line in the template. Tab-completion comes free — `amd set 3 priority <TAB>` would offer the three values without anyone writing bash.

And it's a clean addition, not a migration: **priority doesn't exist yet**. The only trace is a comment at [task.rs:163-165](src/task.rs#L163-L165) anticipating one.

## The ordering inconsistency is nine lines, and it isn't the shell's fault

[render.rs:147-156](src/render.rs#L147-L156):

```rust
fn sort(nodes: &mut [Node]) {
    nodes.sort_by(|a, b| a.task.id.cmp(&b.task.id).then(a.task.stem.cmp(&b.task.stem)));
```

It sorts by id and ignores `order` entirely. Nothing about the terminal makes that hard — the comparator just never got updated when the GUI started writing `order`. Make it `(priority desc, order, id)` and both front ends agree. Worth keeping them as two separate fields, though: **priority is a judgement** (sparse, three values, set deliberately), **order is a position** (dense, fractional, set by dragging). Collapsing them loses the distinction that a high-priority ticket can still sit third in the column.

## You're right about the mechanism — the gap is smaller than you think, and the API is the real problem

The GUI's *entire* write surface is seven calls: `Draft`, `set_title`, `set_points`, `set_body`, `set_epic`, `move_task`, `set_order`, plus `assign` at [gui/mod.rs:235](src/gui/mod.rs#L235). Every one now has a CLI equivalent — your last commit closed most of it. The genuine remaining gaps:

- `amd board` shows one repo; the GUI shows every registered repo. There's no `amd board --all`.
- The sort comparator above.
- Settings (text size, colour scheme) are GUI-only, which is correct.

But here's the finding that actually supports your instinct, and it's an API problem rather than a parity one. The mutators are split across two types with two vocabularies — `Task::set_epic_meta` vs `Board::set_epic`, `Task::set_points` vs `Board::move_task` — and **each one independently re-reads and re-writes the whole file**. So the GUI's save modal does four full read/modify/write cycles for one dialog (assignee, title, points, body). A failure on the third leaves a half-saved ticket, and there's no single place to enforce an invariant like "a sprint refuses an unsized ticket."

That is the thing `amd-lib` should fix: **one mutation vocabulary, one write per user action**. Something closer to `board.update(&task, |t| { t.title = …; t.points = … })?` — atomic, validated in one place, and identical whether the caller is a window or a terminal. Parity then isn't discipline, it's arithmetic: if the lib is the only door, a command that exists is a command both front ends can reach.

## Still open

Two questions from before that decide the shape of the work:

1. **The rule** — is it "amd-lib is the only thing that touches the filesystem and git; it never prompts, never prints, never checks for a terminal"? That would make `board.rs:513`'s `println!` and `render.rs` living in the lib both compile-level violations, which is exactly the enforcement you want.
2. **ADR scope** — agile-md's own internals (→ `docs/adr/` in this repo), or cross-repo standards for the whole mrdoodles family (→ a shared repo, and you're right)?

Separately: the obsolete help text is ten lines in [main.rs:35-41](src/main.rs#L35-L41) and it's wrong in front of users today. Say go and I'll land it as its own commit now rather than carrying it into the rewrite.

## You

we need one library which is the only thing that touches the filesystem, two interfaces that interact with the library, one for the cli and one for the gui. The users will be predominantly using the GUI but some users might like to use the cli and AI will use the CLI. The commands must be consistent  so both the interface must adhere to a common API. does that make sense?

## Claude

Yes, it makes sense — and it's the right shape. Let me state it back precisely, because the rule only holds if the edges are named, and then flag three things that follow from "AI will use the CLI" which the current design doesn't cover.

## The rule, in a form that survives contact

> **`amd-lib` owns all persisted board state. It is the only crate that reads or writes task files, group folders, the registry and the config. It never prompts, never prints, and never asks whether a terminal is attached. The interfaces translate intent into library calls and results into pixels or text.**

Two exceptions have to be written down or someone will "fix" them later:

- **The CLI's `$EDITOR` temp file.** [form.rs](src/form.rs) writes a scratch file to hand a rendered ticket to `vi`/`code --wait` and reads back what returns. That's CLI-owned I/O and it never touches the board — the result comes back as a `String` and goes to the library like any other value.
- **Front-end presentation state.** Text size and colour scheme are the GUI's business. They currently live in the shared [settings.rs](src/settings.rs), which is defensible since it keeps unknown keys, but the ADR should say which side owns which keys.

Everything else — including [board.rs:513](src/board.rs#L513)'s `println!` and [render.rs](src/render.rs) with its `IsTerminal` and OSC-8 escapes — moves to `amd-cli` and becomes a compile error if it drifts back, because `amd-lib`'s manifest simply won't list `richrs`, `clap`, `inquire` or `egui`.

## "Common API" needs a mechanism, or it's just an intention

The version I'd argue for: **the library exposes typed operations, and both interfaces are thin translators.** `amd set 3 points 5` and dragging a card onto a sprint should both bottom out in the same call with the same validation and the same error. Neither interface gets to know a rule the other doesn't — that's what stops "the GUI can do things the CLI can't" from happening again.

That also makes parity *testable* rather than aspirational: if operations are an enumerable set in the lib, a test can assert every one has a CLI route. That test is the thing that actually prevents drift — ADRs record decisions, tests enforce them.

## Three consequences of AI using the CLI

This is the part your description implies but the current code doesn't have:

1. **No structured output exists.** No `--json`, no `serde_json` anywhere. An agent parsing `amd board`'s indented tree is parsing a display format that's explicitly allowed to change. If AI is a first-class consumer, JSON is a first-class output — and it's cheap, because the lib returns typed values and only the CLI's renderer flattens them.
2. **Exit codes are one bit.** Everything failing returns `ExitCode::FAILURE`. An agent can't distinguish "no such ticket" from "not in a git repo" from "sprint refused an unsized ticket" without matching on error strings.
3. **Concurrent writers are now real.** The GUI reloads after its own mutations and has a manual Reload button ([gui/mod.rs:451](src/gui/mod.rs#L451)), but nothing watches the filesystem. So: agent edits ticket 7 via CLI → the open GUI still holds the pre-edit copy → the user saves the edit modal → the agent's change is silently gone. Today that's rare because you're the only writer. With AI on the CLI it's routine. The library is where that gets solved — one write per user action, and a staleness check on read-modify-write — which is another argument for the single mutation vocabulary rather than the current four-writes-per-save.

## What I'd build

```
agile-md/                 workspace
├── crates/
│   ├── amd-lib/          board state, git, templates, groups, registry
│   ├── amd-cli/          clap, inquire, richrs, $EDITOR  → binary `amd`
│   └── amd-ui/           egui/eframe                     → binary `amdui`
└── docs/adr/
```

Binaries stay `amd` and `amdui` — those are installed and documented. And it's an extraction, not a rewrite: the seams are already in the right places, so most of this is moving files and turning `println!`s into return values.

On the ADRs: you asked for a separate `amd-adr` repo and I've argued twice for `docs/adr/` in the workspace, so let me stop arguing and just say what I'll default to — in-repo, because an ADR and the code that violates it can then move in one commit. If they're meant as cross-repo standards for rust-release and conventional-validator too, say so and they go in their own repo; it's a directory move either way.

Shall I draft the plan — crate layout, what moves where, the operation vocabulary, and ADR-0001 (one library, two interfaces) through ADR-0004? I'd rather you approve the operation vocabulary on paper before I start moving 5,000 lines.

## You

There are two sets of settings at present and possibly more at a later version. What we have at the moment is the colour scheme and font size which are managed bt the gui and and repositories. we possibly need two sets of settings, an application wide set of  and gui specific settings, both of these sets of settings require to be saved to disk and both sets need to be passed/availiable to the gui but only the repositories need to be available to the cli

## Claude

Good — and the shape is closer to done than it looks: [settings.rs](src/settings.rs) already preserves unknown keys, and the GUI's values are already namespaced as `gui.theme` / `gui.font`. So this is mostly formalising what's there. One reframing and one consequence, though.

## It's three concerns, not two

Repositories aren't settings. A setting is something the user chose; the repo list is something the *tool accumulated* — [registry::remember()](src/registry.rs#L141) appends to it as a side effect of every command that resolves a board. Different lifecycle, different write frequency, different failure mode.

| | what it is | written by | read by |
|---|---|---|---|
| **app settings** | user preferences, interface-agnostic | user action | CLI + GUI |
| **ui settings** | theme, font size, window geometry | user action | GUI |
| **registry** | accumulated state | automatically, constantly | CLI + GUI |

If you merge the registry into a settings file, then every `amd board` rewrites the file holding the user's preferences. That's a clobber risk for no benefit, and it makes `git`-style "show me my config" useless because it's mostly machine noise. They're already two files (`config` and `repos`) — keep it that way.

Worth noting for the ADR: XDG actually distinguishes these, and `repos` is state, so `$XDG_STATE_HOME/agile-md/repos` is the technically-correct home. Moving it breaks existing installs, so it's either "read the old path as a fallback for a release or two" or "deliberately stay put for compatibility." Either is fine; deciding by accident isn't.

## The consequence of the rule you just set

"Only the repositories need to be available to the CLI" is about *availability*. But you also said the library is the only thing that touches the filesystem — and those two combine into something you may not have intended: **`amd-ui` cannot write its own theme file.**

The resolution is a clean one, and it's the split the code already gestures at:

- **`amd-lib` owns the file** — path resolution, format, atomic write, unknown-key preservation, "a broken config is an empty config, never an error."
- **`amd-ui` owns the schema** — `Theme`, `FontSize`, and whatever comes later. It hands the lib a set of typed key/values under its namespace and asks for them back.

The lib never learns what `nord` means, so UI vocabulary stays out of it, and the rule survives intact. Same mechanism serves the next front end without touching the lib.

## The app tier is currently empty — and that's the contradiction

There's nothing app-wide today. What naturally belongs there is defaults for the knobs that are env-var-only right now: board directory (`AMD_DIR`), branch types (`AMD_BRANCH_TYPES`), auto-register on/off (`AMD_NO_REGISTER`), editor, and which column `amd new` lands in. Every one of those is needed by the CLI — so the app tier isn't GUI-only; it's the *shared* tier, and only the `ui.*` namespace is GUI-only. I think that's what you meant, but the sentence as written says otherwise, and it's the kind of thing that gets built literally.

## Decide precedence now, because an agent will hit it

**flag > env > file > built-in default.** If the file can win, then a user changing a setting in the GUI silently changes what the CLI does on the next agent run, and an agent can't force deterministic behaviour with a flag. This is a two-line rule that's very expensive to retrofit.

## Two small things

- [settings.rs:4](src/settings.rs#L4) still explains itself in terms of "the TUI writing its theme" — the TUI was dropped in `cc713f7`. Same class of staleness as the help text.
- Settings should be **load-modify-save as one operation**, not loaded once and held for the session. A GUI holding a `Config` from startup and saving on exit will clobber anything the CLI wrote in between — the same staleness bug as the ticket editor, just quieter.

This is ADR-worthy in its own right. Shall I fold it in as **ADR-0005: settings, state, and precedence**, alongside the four I offered? And I'd still like the operation vocabulary approved on paper before anything moves.

## You

your plan is correct , This adr is specific to this application so docs/adr works ok. on the app tier perspective. would the app tier be the place that the owning and registered repositories sit? on concurrency perhaps we should also include some locking and release handling so that a ticket becomes locked to others, of course this then requires the capability to be able to unlock a locked ticket. On the precedence front I am unsure at the moment, we need to in more detail. Agreed the load-modify-save is correct. definitely add ADR-0005

## Claude

Yes to all of it. Three answers, then I'll write them up.

**Registry in the app tier — yes as *scope*, no as *storage*.** It's app-level data: both interfaces need it, it's not GUI-specific, so it sits in the shared tier conceptually. But it stays its own file, because `registry::remember()` writes on every command that resolves a board, and preferences are written when a human changes one. Same tier, different file, different write discipline. Worth separating one more thing: the **owning** repository is never stored — it's derived at runtime from `git rev-parse --show-toplevel`, and [Registry::boards()](src/registry.rs#L121) folds it in whether or not it's on the list. Only the *list* is persisted. If we ever store "the current repo" we've created a second source of truth for something `git` already answers.

**Locking — two different problems, and only one of them is locking.** The staleness bug I described (GUI holds a stale copy, saves, clobbers the CLI's write) is a *lost update*, and locks are the wrong tool for it: it needs compare-and-write in the lib — check the file hasn't changed since we read it, fail if it has. Invisible, automatic, no user-facing state, and it fixes the bug even when nobody has locked anything.

What you're describing — "this ticket is mine, don't touch it" — is coordination, and it's a genuine feature, especially with agents on the CLI. But it has to be honest about what it can do: in a git-backed board there's no mutex. A lock is a line in a file, and another machine only learns about it after a pull. So it's an **advisory lease**, not a lock: it stops the *local* tools writing, and it tells a human (or an agent) that someone else has claimed this. I'd put it in the ticket's frontmatter rather than a sidecar, so it travels with the board and shows up in review — consistent with "git is the audit trail" — and give it an expiry, because the honest answer to "who unlocks a lock left by someone on holiday" is "it lapses on its own." Force-unlock exists for the rest.

**Precedence — deferred, as you asked.** ADRs carry a status, so I've written it as `Open` inside ADR-0005 with my recommendation recorded so there's something concrete to argue with, rather than leaving a blank.

## Claude

Written to [docs/adr/](docs/adr/) — seven records plus an index, and a migration plan. No internal links are broken.

| | | Status |
|---|---|---|
| [0001](docs/adr/0001-status-is-the-folder.md) | Status is the folder | Accepted |
| [0002](docs/adr/0002-git-history-is-the-audit-trail.md) | Git history is the audit trail | Accepted |
| [0003](docs/adr/0003-tickets-are-rendered-from-templates.md) | Tickets are rendered from templates | Accepted |
| [0004](docs/adr/0004-one-library-two-interfaces.md) | One library, two interfaces | Accepted |
| [0005](docs/adr/0005-settings-state-and-precedence.md) | Settings, state and precedence | Accepted, precedence open |
| [0006](docs/adr/0006-the-operation-vocabulary.md) | The operation vocabulary | Proposed |
| [0007](docs/adr/0007-concurrency-and-locking.md) | Concurrency and locking | Proposed |

0001–0003 record decisions you'd already made — they're written down so a future change to them is deliberate rather than accidental. 0004 carries the rule verbatim. 0006 and 0007 are Proposed because they're my recommendations, not your decisions yet.

**Three things in there worth your attention before I touch code:**

The **operation table in [0006](docs/adr/0006-the-operation-vocabulary.md)** is the "approve the vocabulary on paper" I asked for. It lists the gaps explicitly rather than hiding them — `amd rm` and `amd link` have no GUI route, `amd templates` has no GUI route, and the GUI's all-repositories view has no CLI equivalent. Branch-switching I've marked as a *deliberate* asymmetry (a GUI has no business checking out a branch under you), but that's a judgement you should confirm.

**[0007](docs/adr/0007-concurrency-and-locking.md) splits your locking request in two.** Optimistic writes fix the lost-update bug for everyone, invisibly, with nothing to remember to release. Leases are the coordination feature you asked for, and I've been deliberate about the name — in a git-backed board a "lock" would promise a guarantee the design can't keep across clones. Four open questions at the bottom, the most interesting being whether `amd start` should take a lease automatically, since it already means "I'm working on this."

**[docs/workspace-split.md](docs/workspace-split.md) has a Phase 0 you won't like:** `tests/test.sh` has to go green *before* anything moves. It's the only safety net for relocating 5,000 lines, and right now it can't tell us whether a move broke something because it's already failing 29 assertions. That's a real piece of work ahead of the fun part, but doing the extraction on a red suite means finding out in a month.

Nothing is committed and no code has changed. Say the word and I'll start on Phase 0 — or Phase 1 first if you want the stale help text gone today, since that's still wrong in front of users.

## You

one thing i want to throw in now so that it doesnt result in a complete rewrite later, maybe some time in the future there might be a requirement for a server component hosted somewhere or github app etc. allowing allowing team working via an agent component within the application. this is just a possibility. I don't think it really changes what we are doing right now though?

## Claude

Mostly right — but there are three decisions that are nearly free today and genuinely expensive to retrofit. Let me check the current shape first.

## Claude

You're right that it doesn't change the plan — but it changes **three signatures**, and those are the ones that would have forced the rewrite. Recorded as [ADR-0008: Room for a server](docs/adr/0008-room-for-a-server.md).

**The three constraints, all cheap now:**

1. **The library never discovers its own context.** `Board::at(path)` is the API; `Board::locate()` — cwd plus `git rev-parse` — is a front-end convenience. Both already exist ([board.rs:86](src/board.rs#L86) and [board.rs:102](src/board.rs#L102)), so this costs nothing: the rule is just that operations take a board and never call `locate()` internally.

2. **Identity is a parameter, not an ambient fact.** This is the expensive one. Today the library reads `git config user.name` from *inside* `Draft::prepare` ([create.rs:145](src/create.rs#L145)). Correct for one person at one terminal; wrong for a server acting for many users, and wrong for an agent acting *on behalf of* someone who isn't the process owner. Retrofitting it changes the signature of every operation that records who did something — and ADR-0007's lease holder depends on it. I've moved it into Phase 5 of the plan, so it lands *with* the new signatures rather than after them.

3. **No process-global state in `amd-lib`.** The only two `OnceLock`s are in `form.rs` and `render.rs`, both already headed for `amd-cli`. Just don't acquire any new ones.

Each is defensible on testability alone, so if the server never happens they cost nothing.

**Explicitly deferred**, and I'd push back if these come up: a storage abstraction trait, async, and anything network/auth-shaped. Async especially — someone always argues for it "in case there's a server", but a server runs blocking filesystem work on a thread pool via `spawn_blocking`. Making the library async now infects every caller and every test to buy something that's already free.

**One tension I've named rather than left to be discovered:** ADR-0002 says `amd` never commits — the user decides when a change is recorded. A hosted component working on its own clone has no working tree to trespass on and must commit to do anything. When that happens, 0002 gets **scoped, not overturned**. I've added a "See also" pointer to [0002](docs/adr/0002-git-history-is-the-audit-trail.md) so whoever hits it finds the reasoning instead of an apparent contradiction.

The reassuring part: [0006](docs/adr/0006-the-operation-vocabulary.md) is already most of a server's groundwork — typed operations, typed failures and `--json` are exactly what an HTTP layer sits on — and [0007](docs/adr/0007-concurrency-and-locking.md)'s compare-and-write doesn't care whether the competing writer is a colleague or a process. A server would be a fourth crate on the same library, additive to the workspace.

Eight ADRs and the plan are on disk, nothing committed, no code touched. Phase 0 (getting `tests/test.sh` green) is still the gate before anything moves — want me to start there?

## You

yes start om the tests

## Claude

Starting Phase 0. First I need to see the whole suite and understand what the columns actually are now — the important part is separating "stale assertion" from "real regression", because the suite is the compatibility contract.

## Claude

Found the second root cause: `junk/` was renamed to `archive/` (and the message from "junked" to "archived"). Let me map every stale assertion precisely.

## Claude

Now the fixes. These are all stale assertions — the behaviour changes (new tickets land in `backlog/`, `junk/` renamed to `archive/`) were deliberate, and the column-flow assertions (`doing/`, `done/`) are still correct and stay untouched.

## Claude

118 passing, 0 failing — and the total is unchanged (89+29 = 118), so nothing was dropped or skipped. Now let me check whether the suite actually covers the features that caused the drift.

## Claude

Serious gap: `amd set` and `amd group` — the commands added in the last two commits — have **zero** coverage. That's what let the drift go unnoticed, and extracting them blind is exactly what Phase 0 is meant to prevent. Let me learn their real semantics before writing assertions.

## Claude

Behaviour confirmed. Two more cases to pin down before I write the assertions.

## Claude

My assertion was wrong, not the code: a fresh ticket has no `points:` key at all — it's added lazily by `amd set`. That's worth pinning down explicitly.

## Claude

Mutation caught by three assertions — the new coverage is real. Restoring and running the full gate.

## Claude

Phase 0 done. **135 passing, 0 failing** — up from 89/29, with 17 assertions added and none removed.

## What the 29 failures actually were

Two deliberate behaviour changes the suite never caught up with — no regressions:

- **`amd new` lands in `backlog/`, not `todo/`.** Creation-site paths only; the column-flow assertions (`doing/`, `done/`) were already correct and I left them alone.
- **`junk/` was renamed `archive/`**, and the message from "junked" to "archived". `amd junk` survives as an alias on `amd rm` ([main.rs:117](src/main.rs#L117)).

One diagnostic caught me out worth mentioning: my instrumented runner re-ran each failing command to capture its output, which made `amd rm` look like it couldn't find the ticket — it had actually archived it on the first run. The real failure was only the changed wording.

## The gap that mattered more

`amd set` and `amd group` — the commands from your last two commits — had **zero** coverage. That's precisely why the drift went unnoticed, and it's the code ADR-0006 says must stay in parity. I've added 17 assertions covering:

- `set points` / `title` / `order`, including that retitling rewrites the title but *not* the filename — the slug and branch stay fixed at creation
- epic and sprint creation, duplicate refusal, `group list` totals
- the sprint rules end to end: refuses an unsized ticket, takes a sized one, and once started gives nothing back and takes nothing more

They live in their own temp repo, because epic directories inside `backlog/` would otherwise disturb the id-sequencing assertions above them.

**I mutation-tested the new coverage** rather than trusting green: disabling the sprint sizing rule in [board.rs](src/board.rs) made three assertions fail, then I restored it. A test that can't fail isn't a safety net.

One spec detail I got wrong first time and pinned down properly: sizing keys are written on demand, so an unsized ticket has *no* `points:` line at all rather than an empty one.

## Also corrected

The ADRs I wrote earlier described the `junk/` drawer — written from CLAUDE.md, which is itself stale on this. [0001](docs/adr/0001-status-is-the-folder.md), [0002](docs/adr/0002-git-history-is-the-audit-trail.md) and [0004](docs/adr/0004-one-library-two-interfaces.md) now say `archive/`.

Full gate is green: fmt, clippy `-D warnings`, `cargo test` (108), shellcheck, and `tests/test.sh`. Nothing committed.

**CLAUDE.md still documents the junk drawer and the `todo/`-on-create behaviour** — that's the same staleness class as the help text, and both are Phase 1. Want me to take Phase 1 next (CLAUDE.md, the obsolete "two ticket types" help text, the mislabelled `clap_complete` comment), or commit Phase 0 first?

## You

yes

## Claude

Reading "yes" as both, in the order I offered: commit Phase 0 first, then Phase 1. Let me check the commit conventions in use.

## Claude

Three commits in, tree clean. Now Phase 1 — the stale surfaces. Let me find all of them rather than just the ones I already know about.

## Claude

Found a real user-facing bug while checking: `--help` documents `AMD_TYPES`, which nothing reads — the live variable is `AMD_BRANCH_TYPES`. And `AMD_NO_REGISTER` is real but undocumented.

## Claude

The env column is now ragged, and there's one more stale claim in there — "covers epics and stories without separate fields" stopped being true when `amd group epic` shipped.

## Claude

Escaping slip — the quotes inside the string literal need escaping.

## Claude

Now the remaining stale surfaces: the TUI references, the `junk` comments, `Cargo.toml`, and the docs.

## Claude

A few more, including the ragged reflow I just caused and an `AMD_TYPES` reference in CLAUDE.md's own env list.

## Claude

Two regressions from my edits. Let me see them.

## Claude

Both failures are tests asserting on the obsolete vocabulary — they were passing *because* the help text was wrong. Let me confirm what the completions actually offer now.

## Claude

These assertions were pinned to the stale help text — `"development"` only ever appeared in the obsolete `--template` description. Repointing them at the real vocabulary.

## Claude

Phases 0 and 1 done, four commits on `rust/gui-board`, nothing pushed. **136 passing, 0 failing**; fmt, clippy `-D warnings`, 108 unit tests and shellcheck all clean, and `--no-default-features` still builds.

I read your "yes" as both — commit Phase 0, then do Phase 1. If you only meant Phase 1, the commits are local and `git reset` undoes them.

## Phase 1 found worse than expected

`amd --help` was actively misinforming people. Beyond the "two ticket types" paragraph you'd already flagged, it **documented `AMD_TYPES`, which nothing has read for some time** — the live variable is `AMD_BRANCH_TYPES`, and that plus `AMD_NO_REGISTER` went unmentioned. `--template` was described as "Ticket type: development (the default) or admin". All corrected, plus a line saying new work lands in `backlog/`.

**The part worth your attention:** two tests were pinned to that wrong documentation. `completions::tests::scripts_complete_the_values_too` asserted the fish script contains `"development"` and `"feat"` — which passed *only because the help text was wrong*. Fixing the docs broke them. They now assert on values a parser actually advertises: branch types, templates, columns and the `set` fields.

That's the same failure mode as the stale suite, one level down: a test that passes because two wrong things agree with each other. Worth remembering when we get to ADR-0006's parity test — it has to assert against the library's operation set, not against whatever the CLI currently prints.

The rest was mechanical: `junk` → `archive` throughout (`Board::junk` is `Board::archive` now), TUI references in `settings.rs`/`gui/settings.rs` (removed in `cc713f7`), `clap_complete` commented "GUI only" when it generates completions, `branch.rs` described as mapping commit types to branch prefixes, `render.rs` described as drawing tables when it draws a tree.

CLAUDE.md gained the two facts whose absence let the suite drift in the first place: the columns are `backlog`, `todo`, `doing`, `done`, and `amd new` lands in `backlog/`.

## Where that leaves the plan

Phases 0 and 1 are complete. **Phase 2** is next — introduce the workspace with `amd-lib` only, binaries staying put and depending on it. Nothing moves, so it should be a pure build-system change with the suite as the check.

One thing I'd like your call on before Phase 2: **`amd-lib` will need a public name**. The crate is `agile-md` on the library side (`agile_md::` in every `use`), and renaming it churns every import in the codebase. I'd keep the package `agile-md` with `[lib] name = "amd_lib"` only if you actually want the new name in `use` statements — otherwise leaving the library as `agile_md` costs nothing and keeps the diff to Cargo.toml files.

## You

leave the library as agile_md

## Claude

Understood — `agile_md` stays. Let me work out the dependency split before moving anything.

## Claude

`time` and `tempfile` matched too loosely — let me check them precisely.

## Claude

Clean split confirmed — `richrs` is only in `render.rs`, and `tempfile` is test-only everywhere except [form.rs:82](src/form.rs#L82), which is exactly the `$EDITOR` exception ADR-0004 names. Doing the full three-crate skeleton now rather than one crate, so the remaining phases are pure content moves.

## Claude

It builds, but with a warning that matters: `amd gui` would silently become "this build has no desktop board". Phase 2 is meant to be behaviour-preserving, so the CLI keeps a `gui` feature.

## Claude

All green. Now the docs that describe the old layout.

## Claude

`--no-default-features` at the root no longer gives a headless build — `amd-ui` forces the feature on. The headless build is now `-p amd-cli`. Updating the docs to match.

## Claude

Now CLAUDE.md's Layout section, which points at paths that no longer exist.

## Claude

Phase 2 done and committed. The workspace is real:

```
crates/amd-lib/   the library (agile_md) — everything that touches the board
crates/amd-cli/   → binary `amd`
crates/amd-ui/    → binary `amdui`
```

21 files moved with `git mv`, **not one line of Rust changed**, and `git log --follow` still reconstructs `board.rs`'s 16 commits. 136 end-to-end assertions and 108 unit tests pass unchanged — which is the return on doing Phase 0 first.

I did more than the plan's Phase 2 said ("workspace, one crate"): all three crates exist now. Standing up the skeleton in one go makes every remaining phase a pure content move rather than another round of manifest churn.

**The boundary is now enforced by the manifest.** `amd-lib` doesn't list `clap`, `inquire` or `libc` — a front-end dependency reaching into the library fails to compile. It still lists `richrs` and `egui`, because `render.rs` moves in Phase 3 and `gui/` in Phase 4; the manifests say so explicitly rather than leaving it to be discovered.

**Two decisions I made along the way, both worth your eye:**

`amd-cli` keeps a `gui` feature, on by default. Without it, `amd gui` would have silently degraded to "this build has no desktop board" — the compiler warned me. Making `amd gui` exec the `amdui` binary would take egui out of `amd-cli` entirely, which is architecturally nicer, but a prebuilt install ships only `amd`, so those users would lose the board completely. That trade belongs to packaging, not to a mechanical phase — noted in the plan.

**One behaviour change you should know about:** `cargo build --no-default-features` at the workspace root no longer gives a CLI-only build. `amd-ui` requires `agile-md/gui`, and feature unification turns it back on. The headless build is now `cargo build -p amd-cli --no-default-features` — verified working, and README, CLAUDE.md and the plan all say so.

**Next is Phase 3**: `render.rs` moves to `amd-cli`, and the library stops printing — `board.rs`'s `println!("moved …")` becomes a returned value. Unlike Phase 2 this one touches real code, so it's the first phase where the suite is doing more than confirming a file move. Want me to carry on?

## You

yes carry on

## Claude

Phase 3 touches real code, so let me map the dependencies first — particularly whether the GUI uses anything in `render.rs`.

## Claude

`render` is used only by the CLI — the GUI's `flatten` hits are iterator methods. It moves wholesale. Let me see how the callers of `move_task` behave.

## Claude

No assertion on the `moved` line — I'll preserve the wording exactly and add one, since it becomes a CLI contract in this phase.

## Claude

The assert tripped on the word in a comment — neither manifest was written. Redoing it.

## Claude

Now the part that changes behaviour: the library stops printing.

## Claude

The move revealed dead code that `pub` was hiding in the library. Let me see how much.

## Claude

`flatten` was written for the TUI's list widget — dropped in `cc713f7`, no callers since. Removing it.

## Claude

Phase 3 done. **`amd-lib` is now silent** — verified, not assumed: no `println!`, no `IsTerminal`, no process-global state outside `gui/`, and `richrs` is gone from its manifest, so the library couldn't draw a board even if someone tried.

**The `println!` was worse than a style problem.** `Board::move_task` printed `moved X -> done/` itself — which meant every card you dragged in the desktop board wrote a line to a stdout nobody was reading. It now returns where the ticket landed, and `amd` prints the identical line from a `move_and_report` helper. The wording is unchanged, and the suite pins it now that it's a CLI contract.

A side effect worth noting: ADR-0008's third constraint (no process-global state in `amd-lib`) is satisfied **for free** — both `OnceLock`s lived in `render.rs` and `form.rs`, and both are now in `amd-cli`.

**Moving the file out revealed dead code.** `render::flatten()` — "the shape a list widget wants" — was written for the TUI, has had no caller since `cc713f7` removed it, and stayed invisible because `pub` in a library makes "unused" unprovable. The compiler found it the moment it became a private module in a binary. Removed. That's an argument for the split that I hadn't anticipated: the boundary doesn't just prevent bad dependencies, it makes the dead code visible.

137 end-to-end assertions and 108 unit tests pass. The render tests travelled with the file from `amd-lib` to `amd-cli`.

## What's left

**Phase 4** — move `gui/` to `amd-ui`, splitting `gui/settings.rs` into schema (`amd-ui`) and storage (`amd-lib`) per ADR-0005. After that `amd-lib` lists no `egui` at all and ADR-0004's rule is fully enforced by the manifests. It's the biggest single move left (~1,600 lines) but structurally the same shape as Phase 3.

One thing to decide when we get there, which I'd rather flag now than discover mid-move: **`amd gui` currently keeps `egui` inside the `amd` binary** via `amd-cli`'s `gui` feature. Once the board lives in `amd-ui`, `amd-cli` would have to depend on `amd-ui` to keep that working — a CLI crate depending on a GUI crate, which is backwards. The clean answer is `amd gui` execing `amdui`, but that only works if the release ships both binaries, and `rust-release` packages one. So Phase 4 probably forces the packaging question you've been able to defer.

Want me to carry on into Phase 4, or sort the packaging first so Phase 4 has somewhere to land?

## You

Sort out the packaging first

## Claude

Let me investigate `rust-release` properly before proposing anything — including who else depends on it.

## Claude

`rust-release` is its own repo (`mrdoodles/rust-release`), pinned at `@v1`, and agile-md is its only consumer among your local clones. So the change is small — but it's a different repository and it means moving the `v1` tag.

## Claude

There's uncommitted work in that repo — let me see it before touching anything.

## Claude

Their in-flight work is a separate concern (generated files), so I'll keep it out of my commit. Now the change itself.

## Claude

Now let me actually verify the logic rather than trust it — simulating the step against fake build outputs.

## Claude

Case 5 exits 1 but prints nothing — `set -o pipefail` kills the script at the failing `grep` before my error message runs. Fixing that.

## Claude

Both edge cases now behave. Updating `rust-release`'s docs to match, keeping clear of their in-flight paragraph.

## Claude

Now committing in `rust-release` — staging only my own changes so their in-flight README paragraph stays uncommitted.

## Claude

Now the agile-md side.

## Claude

`install.sh` already handles a zip with both. Updating the comments that said otherwise, then testing the install path for real.

## Claude

Packaging sorted. Both sides are committed locally, nothing pushed.

**In `mrdoodles/rust-release`** — `bin-name` now takes a space- or comma-separated list, staging every named binary into one zip. Backward compatible: a single name behaves exactly as before. The zip keeps the name of the **first** binary, so `amd-<target>.zip` is unchanged and nobody's existing `curl` breaks.

I verified it against a simulated build tree rather than trusting it, and **the simulation caught a real bug**: with no `bin-name` and a virtual workspace manifest, the step exited 1 silently. `grep` returns 1 when there's no `[package] name`, and `set -o pipefail` killed the step before my error message could run. It now says *"a virtual workspace manifest needs bin-name"* — which matters, because that's precisely the layout that has several binaries. Six cases pass: one binary, two space-separated, two comma-separated, a missing binary (fails naming it), a virtual manifest, and a normal manifest still deriving its name.

Their own suite (9 assertions) and shellcheck are clean. **I kept their in-flight work out of my commit** — there's an uncommitted README paragraph and `docs/packaging-generated-files.md` about packaging man pages/completions, still sitting in the working tree untouched.

**In agile-md** — `bin-name: "amd amdui"`. `install.sh` already copied `amdui` when present, so only its comments needed correcting; the `if [ -f ... ]` guard stays deliberately, since `--version` can pin a release made before the zip carried both. Verified by running `install.sh --from-source` into a temp dir: both binaries installed, both answer `--version`.

## One thing left, and it's yours

**`release.yml` pins `rust-release@v1`, so nothing changes until the `v1` tag moves.** That means pushing `mrdoodles/rust-release` and force-moving `v1` — outward-facing, and it affects any consumer I can't see from here (only agile-md among your local clones, but I can't check GitHub). I've left both, along with agile-md's push. Say the word and I'll do it, or you may prefer to handle the tag yourself.

Phase 4 now has somewhere to land: with `amdui` in every install, `amd gui` can exec it and `amd-cli` can drop `egui` entirely. Worth noting that becomes a hard ordering dependency — **Phase 4 shouldn't ship in a release before the `v1` tag moves**, or `amd gui` would exec a binary that isn't there. Want me to carry on into Phase 4 on that understanding, or wait until the tag is out?

## You

<ide_opened_file>The user opened the file /Users/mrdoodles/develop/github/mrdoodles/agile-md/amd-rust/.github/workflows/release.yml in the IDE. This may or may not be related to the current task.</ide_opened_file>
we also have pr 13, is this still required ?

## Claude

API Error: 529 Overloaded. This is a server-side issue, usually temporary — try again in a moment. If it persists, check https://status.claude.com.

## You

<ide_opened_file>The user opened the file /Users/mrdoodles/develop/github/mrdoodles/agile-md/amd-rust/.github/workflows/release.yml in the IDE. This may or may not be related to the current task.</ide_opened_file>
This is getting there but we are not release ready yet. In the backlog view we should see a list of the current epics and a list of current sprints so perhaps it is three colums | Unfiled | Epics | Sprints | ?

## Claude

API Error: 529 Overloaded. This is a server-side issue, usually temporary — try again in a moment. If it persists, check https://status.claude.com.

## You

<ide_opened_file>The user opened the file /Users/mrdoodles/develop/github/mrdoodles/agile-md/amd-rust/.github/workflows/release.yml in the IDE. This may or may not be related to the current task.</ide_opened_file>
is pr 13 still required

## Claude

Let me look at PR 13.

## Claude

`amd-rust` *is* the repo root — so `main` holds the bash line and `rust/gui-board` holds everything we've been doing. Let me see PR 13's actual diff.

## Claude

CI passes; it's blocked only by a required review. And `origin/main` still presents the bash tool with no notice.

**Yes, PR 13 is still required.** Nothing in the last two days touches `main` — all our work is on `rust/gui-board`, unpushed. A reader landing on the repo today gets the bash README with no signal it's frozen, and the trap it documents ("boards are compatible, commands are not — `amd doing` is `amd start` there") is *more* true now, not less: we've since added `amd set`, `amd group`, epics and sprints.

Two caveats. It's a **signpost with a defined lifetime** — the moment the Rust line lands on `main`, both the notice and the `rust/gui-board` link become wrong, so it wants revisiting then rather than being left to rot. And its *body* says "13 commits" where there are now ~19; that's PR description text, not the diff, so it doesn't affect what merges.

Now the backlog view — let me see what it does today before I opine.

## Claude

Now the backlog view. What's there today is one column *per group*: `| Unfiled | epic-a | epic-b | sprint-1 | sprint-2 | …`, growing horizontally without limit, with epics and sprints indistinguishable from each other ([gui/mod.rs:306](crates/amd-lib/src/gui/mod.rs#L306)).

**Your three columns are right**, and for a better reason than tidiness: epics and sprints obey *different rules*, and the current view hides that. A sprint refuses an unsized ticket and, once started, refuses everything; an epic accepts anything. Putting them in one undifferentiated row means the drop rules look arbitrary. Separate columns make them legible.

Two things it has to get right, or it'll be worse than what's there:

**The columns hold different kinds of thing.** Unfiled holds tickets; Epics and Sprints hold *containers*. So the drop target becomes a card inside a column rather than the column itself, and — more importantly — an epic's tickets are no longer visible. File a ticket and it vanishes. Each epic/sprint card needs to expand in place (egui's `CollapsingHeader` does this) showing its tickets, count, and points total. Otherwise the view is write-only.

**A refusal must be visible.** Drag an unsized ticket onto a sprint today and the error goes to the status line *behind* the modal — your ticket 025, which CLAUDE.md already describes as "a failure reads as 'the button does nothing'". In the current view that's an annoyance. In a three-column view where sprints are a distinct drop target, silent refusal is the single most likely interaction, and it would read as the feature being broken. **025 stops being a nice-to-have and becomes a prerequisite.**

Worth showing per sprint: state (pending/started), days, and the points total — that last is the number a sprint exists for, and `amd group` already prints it.

## On sequencing

The backlog view lives in `crates/amd-lib/src/gui/`, which Phase 4 moves wholesale to `amd-ui`. Redesigning it first means doing substantial work in the crate it's leaving, and turning a clean `git mv` into a move-plus-rewrite that's much harder to review. I'd do **Phase 4 first** (it's a pure relocation, no behaviour change), then build the three-column backlog in `amd-ui` where it will actually live.

That also fits the release question: Phase 4 needs the `v1` tag moved before anything ships, since `amd gui` will exec `amdui`. So the order I'd suggest is — Phase 4, then push `rust-release` and move `v1`, then the backlog redesign and 025 together, then look at release readiness.

Want me to start Phase 4 on that basis? And separately: shall I leave PR 13 for you to approve, or would you rather I update its body to reflect where the branch has got to?

## You

just a thought, an aside for further down the line but it might be worth consideration now is a sprint view of the currently active sprint displaying burnup and burndown and the actual vs forcast points.

## Claude

Good instinct to raise it now rather than later — the question is whether it forces any decision today. Let me check what a sprint actually records.

## Claude

The `_group.md` format has something worth catching now — and a claim I want to test before I say it.

## Claude

That's a real bug, and precisely the one that would poison a burndown. One more check.

## Claude

Worth considering now — it surfaced two live bugs, both of which would silently corrupt exactly those charts.

## A sprint loses its points as work progresses

```
group list, both tickets in the sprint:   2 ticket(s), 8 point(s)
amd start 001 --no-branch
group list again:                         1 ticket(s), 3 point(s)
```

Nothing was completed. Ticket 001 moved to `doing/` and dropped out of its own sprint's total — [cmd_group](crates/amd-cli/src/main.rs) counts `tasks_in(Column::Backlog)` only. The ticket still carries `epic: "s1"`, so the data is intact; the query is wrong. **The one number a sprint exists for is wrong the moment anyone starts work**, and a burndown built on it would show points vanishing rather than burning down. I'd call that release-blocking on its own.

## `amd rm` walks a ticket out of a started sprint

`set_epic` refuses to move a ticket out of a started sprint — "its tickets are fixed". `amd rm` archives it with no complaint at all. So scope *can* shrink, silently, through the one door nobody guarded.

That matters more than it looks: **the fixed-scope rule is what makes a forecast line meaningful**. If scope can't change, the ideal line is well-defined from day one. It's also why burnup would be near-redundant here — burnup exists to show scope creep, and your design forbids it. Burndown plus a marker when scope *did* change is probably the honest chart, rather than both.

## The one thing that's lossy if deferred

There's no sprint start date. `_group.md` has a field called `started:` — but it holds `"pending"` / `"started"`, a state, not a date:

```
days: "10"
started: "started"
```

Without a date you cannot place a sprint on a calendar, so no x-axis. It can't be reconstructed reliably later either, and every sprint started before the field exists loses its date permanently. That's the ADR-0008 pattern again: cheap now, lossy forever.

It also wants renaming — `state:` for the state, `started:` for the date — with a fallback read for boards already written, which is the pattern `Task` already uses for its old keys.

## The good news

Completion times need **no new state**. Git already has them:

```
$ git log --follow --diff-filter=R --date=short --format='%ad %s' -- tasks/done/002-two.md
2026-08-18 finish 002
2026-08-18 start 002
```

Every column transition, dated. That's ADR-0002 paying for itself — a burndown is a query over history, not a feature that needs a store. The one caveat to write down: it's only as good as the user's commit discipline, since `amd` never commits.

## What I'd do

Fix the counting bug and add the start date **now** — both small, and the second is the only irreversible one. Defer the charts entirely; they need nothing further once those two exist.

The open question is your call: **should `amd rm` refuse to archive a ticket out of a started sprint?** Refusing is consistent with "gives nothing back", but it leaves no escape hatch for a ticket raised in error. The alternatives are allowing it with a warning, or requiring `--force`. That decision determines whether a sprint's scope is genuinely immutable — which is the premise the whole forecast rests on, so it's worth an ADR rather than a default.

## You

I have been in teams where tickets have been moved out, renamed and very rarely deleted from an active sprint, its not good practice and messes the charts but particularly when a team is new it happens. I would say that tickets can be changed, moved in and out but not archived. However a ticket moved out could be archived,

## Claude

That's a clear rule, and a better one than what's there — because it makes the two cases differ in what they leave behind. A ticket moved out of a sprint is a `git mv`: the scope change is in the history, dated, with the points still readable on disk. A ticket archived from inside the sprint is gitignored and `git rm --cached`'d, so the board no longer shows that those points ever left. Forbidding the second forces the first, and the chart stays reconstructible.

It also changes my advice from last turn: **with scope genuinely mutable, burnup stops being redundant.** Your original instinct to ask for both was right.

Let me look at the code that enforces the current rule.

## Claude

Now the counting bug — a sprint should keep its tickets wherever they've got to.

## Claude

The membership query lives in the CLI, and the GUI has its own copy — exactly what ADR-0006 says shouldn't happen. Putting it in the library. First, is `epic` read from frontmatter regardless of column?

## Claude

Deeper than it looked: `Task::epic` is derived from the **folder**, and `doing/` has no epic sub-folders — so a ticket loses its epic the moment it leaves the backlog, even though the frontmatter key still carries it.

## Claude

All three rules behave, and the sprint now holds 8 points through start *and* completion. Updating the assertions that encoded the old rule.

## Claude

Implemented and committed. **141 assertions, 109 unit tests, all green.**

## The rule now

A started sprint takes tickets in and lets them out. It refuses exactly two things: an **unsized ticket** (the total is the one number a sprint is for) and an **archive straight out of the sprint**:

```
$ amd rm 002
amd: 002-two is in s1, which has started — move it out of the sprint before archiving it
$ amd set 002 epic ''   &&  amd rm 002
002 filed under (cleared)
archived 002-two.md -> archive/
```

Recorded as [ADR-0009](docs/adr/0009-sprint-scope-and-archiving.md). The asymmetry isn't about how drastic the actions feel — it's about what survives. A move out is a dated `git mv` with the points still readable on the board; an archive is `git rm --cached` into a gitignored directory. A burnup can draw the first and cannot honestly draw the second.

One consequence worth flagging: **`State::Started` is now a moment, not a lock.** It's the origin of the chart's x-axis and the baseline for the ideal line, and `accepts_changes()` is consulted by `archive` alone.

## The counting bug was deeper than "wrong query"

`Task::epic` is derived from the **folder**, and `doing/` has no epic sub-directories — so a ticket lost its epic entirely the moment work started, even though the frontmatter still said `epic: "s1"`. Fixed in the library as `Board::tasks_in_group`, which scans every column and treats the two carriers separately: inside `backlog/` the folder decides, outside it the frontmatter key is all there is. Both front ends had their own copy of that query; now neither does, per ADR-0006.

A sprint now holds 8 points through start *and* completion, where it previously fell to 3.

This is marked `fix!` — breaking. `amd set <ref> epic <name>` no longer errors on a started sprint, so any script relying on that refusal will now succeed instead, and `amd rm` errors where it didn't.

## Still outstanding

**The sprint start date.** `_group.md` has a field called `started:` holding `"pending"`/`"started"` — a state wearing a timestamp's name — and there's no date anywhere, so there's no x-axis. This is the lossy one: every sprint started before the field exists loses its date permanently.

The fix wants a rename (`state:` for the state, `started:` for the date) with a fallback read, which `Task` already does for its old keys. Small, but it changes the `_group.md` format, so I'd rather you nodded at it than found it. Shall I take that next — or Phase 4 first, since the GUI move is still queued behind it?

## You

A thought and perhaps a ticket: when someone is working on a ticket and they hit a problem then if there was an engineering notes section at the bottom of the ticket where they could add their notes then when the ticket is closed the content of the engineering notes could be written out as markdown to file with a suitable title. there could be an engineeering directory where all these notes get placed which allows users to find information and answers to problems. do you think this is a good idea?

## Claude

Yes — with one part of it I'd change, and one part that's free today.

## The section is free

[ticket.md.jinja](crates/amd-lib/templates/ticket.md.jinja) already has `## Notes` and `## Acceptance criteria`. Adding `## Engineering notes` is a template edit and nothing else — ADR-0003 means the template *is* the form, so the capture half of your idea needs no code at all. Any board can have it today with `amd templates eject ticket`.

## The extraction is where I'd push back

Three problems, in increasing order of how much they matter.

**Writing a copy is the one thing this codebase has consistently refused.** No `reporter` field because `git log` has it; no `moved-at` timestamps because the rename has it; ADR-0002 exists to say so. A closed ticket lives in `done/` forever and is tracked. Copying its notes into a second file means two files with the same words, and the moment anyone edits either, they disagree — with nothing to say which is right.

But findability is a genuinely different need from storage, and that's the real argument for your idea. Ticket files are named for the *task* — `004-fix-login.md` — so what you learned is filed under a name that doesn't describe it. That's a fair problem and grep isn't a great answer to it.

The reconciliation: make the export **derived and regenerable**, not a one-time copy. `amd notes export` rebuilds the directory from the tickets every time it runs — a build artifact, not a second source. Drift becomes impossible because it's overwritten, and the ticket stays authoritative.

**"On close" is the wrong trigger.** Notes matter most while someone is *stuck*, not after they've finished. And plenty of tickets never reach `done/` — they get archived or abandoned, and those often hold the most valuable "we tried this, it didn't work, here's why". A close-triggered export loses exactly the ones worth keeping. A regenerable export sidesteps this: it can include every ticket in any column.

**The title is the hard part, and it's most of the value.** "Fix login" tells a future reader nothing. A useful note is titled for the *finding* — "wgpu surface lost after macOS sleep" — which only a person can write, and asking for it at close is friction at the exact moment someone wants to be done. I'd solve it in the template: `### <heading>` per note inside the section, so one ticket can yield several notes and each carries its own title, written when the thought was fresh.

## Two smaller calls

**Where it lives.** `amd` runs in *consuming* repos, so this creates a directory in every repo it touches. I'd put it under the board — `tasks/engineering/` — so a board stays self-contained and `AMD_DIR` relocates the whole thing. Committed and searchable, not gitignored; being findable is the entire point.

**What it actually is.** This is ADR's little sibling: ADRs record decisions, these record findings. Same durability class, much lower ceremony. Worth keeping that framing when deciding the format, because it argues for one file per finding rather than one per ticket.

---

So: good idea, and I'd build it as **template section now, regenerable export later** — never as a side effect of closing.

Happy to raise it as a ticket on this repo's board once you've picked the shape, since a ticket recording the close-trigger design would be worse than none. Which do you want — the export as I've described it, or the simpler `amd notes` that indexes and prints without writing files at all?

Separately, something I noticed while looking: this repo's board has **both `tasks/archive/` and a leftover `tasks/junk/`** from the rename. Want me to fold the old one in?

## You

yes raise it as a ticket. Another possible enhancement would be to add a fuzzy searchthat searches the documents somewhere too ?

## Claude

The leftover `junk/` is empty apart from its `.gitignore`. Now the tickets — raising them with `amd` itself.

## Claude

Both raised, linked and committed — [026](tasks/backlog/026-engineering-notes-on-a-ticket-exported-as-a-findab.md) and [027](tasks/backlog/027-search-across-every-board-from-the-cli-and-the-des.md), dogfooded through `amd new` and `amd link`.

## On fuzzy search — yes, and it partly answers 026

`grep -r` already works in one repo, so the value is in what grep can't do: **search every registered board without knowing the paths**, a filter box on the desktop board, and results that come back as ticket ids you can act on rather than file paths.

One distinction I'd hold firmly, because getting it wrong makes search feel broken:

**Fuzzy matching is right for short strings and actively bad for prose.** Subsequence matching is what makes `amd show chec` find `checkout-revamp` — excellent on ids, slugs and titles. Run the same matcher over a paragraph and nearly everything matches, with noise ranked above signal. Bodies want substring or word matching, ranked by hit count and by where the hit landed, with a title hit beating a body hit. Two different matchers, deliberately.

It belongs in **amd-lib** — both front ends need the same results in the same order, and ranking is part of the result, not presentation. And no index: markdown changes under you on every checkout and pull, which is exactly why the registry stores paths and nothing else. Search reads from disk; the desktop board already holds loaded cards, so a filter can work over those.

**The interesting bit is how it lands on 026.** The whole reason to export engineering notes to files was findability — the ticket is named for the task, so what you learned is filed under a name that doesn't describe it. If search covers ticket bodies across boards, the notes are findable where they already are, and the export stops being worth building. That resolves my drift objection for free rather than by argument.

So I've sequenced them: **027 first, then decide whether 026's export is still needed.** The template section in 026 is independent and can be done any time — it's a template edit, no code.

Still on the table from earlier, and neither is urgent: the sprint start date (the lossy one — no x-axis for a burndown until it exists, and every sprint started before it loses its date), and Phase 4. Plus that empty `tasks/junk/` left over from the rename, whenever you want it gone.

## You

yes get rid of junk

## Claude

Empty, and no code names it as a directory — only the deliberate `amd junk` command alias on `amd rm`.

## Claude

Gone. Board reads fine, 141 assertions still pass, tree clean.

Nothing was lost — it held only its `.gitignore`, no tickets. `amd junk` survives as an alias on `amd rm`, since that's a command name rather than a path.

One thing worth knowing, though it doesn't affect this repo: **other boards created before the rename will still have a `junk/` directory, and `amd` no longer looks there.** Anyone who junked tickets pre-rename has them sitting in a folder the tool ignores. The impact is small — junk contents were always gitignored and off the board, so those files were already invisible to everything except `ls` — but there's no migration, and someone will eventually notice a stray directory. Options if you care: have `Board::ensure` fold an empty `junk/` away silently, or just note it in the README. Not urgent, and arguably not worth code.

## Where things stand

Committed on `rust/gui-board`, nothing pushed:

- Workspace split through Phase 3 — `amd-lib` is silent and draws nothing
- Sprint rules per ADR-0009, and the points-vanishing bug fixed
- Packaging: both binaries in one zip (needs the `v1` tag moved)
- Nine ADRs, the migration plan, and tickets 026/027

**Open, in the order I'd take them:**

1. **Phase 4** — `gui/` to `amd-ui`, after which ADR-0004's rule is fully enforced by the manifests. Pure move.
2. **The `v1` tag** — yours to push; Phase 4 shouldn't ship in a release before it, or `amd gui` execs a binary that isn't installed.
3. **Sprint start date** — the lossy one. No x-axis for a burndown until it exists, and it needs the `started:`/`state:` rename with a fallback read.
4. **Ticket 025 + the three-column backlog** — 025 first, since a silent refusal would make the new view read as broken.
5. **PR 13** — still required, blocked only on your review.

Say which and I'll pick it up.

## You

<ide_opened_file>The user opened the file /Users/mrdoodles/develop/github/mrdoodles/agile-md/amd-rust/.github/workflows/release.yml in the IDE. This may or may not be related to the current task.</ide_opened_file>
I need to install an os update, can you save the transcript to disk  and is there anything else that needs to be done before I start the update ?

## Claude

Let me check what's actually at risk first, then save the transcript.

## Claude

`rust-release` came back clean — let me verify that properly, since I left uncommitted work there.

## Claude

The repo has moved on — different branch, and commits I don't recognise. Let me find my packaging commit.

## Claude

My `rust-release` commit is gone — and the repo has moved org (`mrdoodles` → `lite-actions`). Let me see if it's recoverable.

## Claude

Now the transcript.
