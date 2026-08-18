# TODO — `tms` tmux Session Manager

## Vision

Turn `tms` from a useful personal tmux sessionizer into a robust, fast, project-aware tmux session manager.

## Status — implemented

The reliability, persistence, core project configuration, CLI, and basic FZF
preview milestones are now implemented in `tms-core`.

- [x] Idempotent restore of sessions, windows, panes, indexes, and layouts.
- [x] Versioned, atomically written state with restrictive permissions.
- [x] User-managed search paths in `$XDG_CONFIG_HOME/tms/config`; a default
  configuration is created with `TS_SEARCH_PATHS=("$HOME")` when absent.
- [x] Opt-in `.tmux-sessionizer` support with `tms_window` and `tms_pane`.
- [x] Positional projects, `shell-<directory-hash>` current-directory sessions,
  subcommands, legacy option aliases, confirmation-aware kill, and debug mode.
- [x] FZF previews for Git directories and tmux sessions.

The unchecked items below are remaining work, not descriptions of the current
implementation. Features deliberately not implemented include restoring arbitrary
pane commands, automatic execution of untrusted project config, and removal of
stale windows during restore.

Current strengths:

* FZF-based project/session selection
* Automatic tmux session creation
* Switching between sessions from inside tmux
* Save/restore support
* Project-local `.tmux-sessionizer`
* Configurable search paths
* `--prepare` workflow
* Direct session creation
* Session listing and cleanup

Main areas to improve:

1. Correctness
2. State persistence
3. Project configuration
4. Pane/window/layout restoration
5. FZF UX
6. CLI ergonomics
7. Maintainability
8. Safety and error handling

---

# P0 — Correctness & Reliability

## [x] Make `restore` idempotent

Current behavior can duplicate windows every time `tms --restore` is executed.

Example:

```text
dev
git
test
```

After a second restore:

```text
dev
git
test
git
test
```

### Desired behavior

Running:

```bash
tms --restore
```

multiple times should produce exactly the same tmux state.

Rules:

* [ ] If session does not exist, create it.
* [ ] If session exists, don't recreate it.
* [ ] If window exists, don't recreate it.
* [ ] If window is missing, create it.
* [ ] Preserve window names.
* [ ] Preserve window indexes where possible.
* [ ] Optionally reconcile stale windows later.

---

## [x] Replace `pgrep tmux` with tmux-native checks

Current:

```bash
tmux_running=$(pgrep tmux)
```

This checks for a process rather than whether the tmux server is actually usable.

Prefer tmux-native checks such as:

```bash
tmux has-session
```

or:

```bash
tmux list-sessions
```

### Goals

* [ ] Remove dependency on `pgrep`.
* [ ] Distinguish "tmux process exists" from "tmux server is usable".
* [ ] Avoid unnecessary process inspection.

---

## [x] Simplify `has_session`

Current implementation:

```bash
has_session() {
    tmux list-sessions | grep -q "^$1:"
}
```

Replace with:

```bash
has_session() {
    tmux has-session -t "$1" 2>/dev/null
}
```

### Benefits

* [ ] No regex matching.
* [ ] Correct handling of special characters.
* [ ] Less process spawning.
* [ ] Uses tmux's native API.

---

## [x] Eliminate unsafe regex matching

Current code contains patterns such as:

```bash
grep -q "^$1:"
```

and:

```bash
grep -v "^${session}\t"
```

Session names can contain characters meaningful to regular expressions.

### TODO

* [ ] Avoid regex matching wherever possible.
* [ ] Use `tmux has-session`.
* [ ] Use `awk -v` for session-file filtering.
* [ ] Escape values when regex is unavoidable.

---

## [x] Make `--save` atomic

Current:

```bash
tmux list-windows ... > ~/.tmux-session
```

A failed/interrupted write can leave a corrupt or incomplete state file.

### Desired implementation

1. Create temporary file.
2. Write state to temporary file.
3. Validate it.
4. Atomically move it into place.

Example concept:

```bash
tmp="$(mktemp)"
trap 'rm -f "$tmp"' EXIT

tmux list-windows ... > "$tmp"
mv "$tmp" "$TMUX_SESSION_FILE"
```

### TODO

* [ ] Use `mktemp`.
* [ ] Write to temporary file.
* [ ] Only replace existing state after successful write.
* [ ] Set restrictive permissions.
* [ ] Consider `install -m 600`.

---

## [x] Handle missing session state file

Commands should behave sensibly when:

```bash
~/.tmux-session
```

doesn't exist.

### TODO

* [ ] `--get` should not produce an ugly error.
* [ ] `--restore` should gracefully report "nothing to restore".
* [ ] `--remove` should handle missing storage.
* [ ] `--save` should create the file automatically.

---

## [x] Handle empty tmux server

Commands like:

```bash
tmux ls
```

can fail when no server exists.

### TODO

* [ ] Avoid treating "no tmux sessions" as a fatal error.
* [ ] Make `--ls` return cleanly with no sessions.
* [ ] Make `--save` handle zero sessions.
* [ ] Make `--restore` start the server when needed.

---

## [x] Fix non-TTY restore behavior

Current:

```bash
stty size
```

can fail or return unusable information outside an interactive terminal.

### TODO

* [ ] Detect whether stdin/stdout is a TTY.
* [ ] Only calculate terminal dimensions when appropriate.
* [ ] Allow tmux to choose dimensions when no TTY is available.
* [ ] Avoid generating malformed `-x/-y` arguments.

---

# P1 — Session State

## [x] Introduce a proper state-file variable

Instead of hardcoding:

```bash
~/.tmux-session
```

define:

```bash
TMUX_SESSION_FILE="${TMUX_SESSION_FILE:-$HOME/.tmux-session}"
```

### TODO

* [ ] Replace every hardcoded reference.
* [ ] Allow users to override the location.
* [ ] Document the environment variable.

---

## [x] Define configurable search paths

Current:

```bash
TS_SEARCH_PATHS=("$HOME/workspace" "$HOME/Desktop")
```

Make this configurable.

Possible model:

```bash
TS_SEARCH_PATHS=(
    "$HOME/workspace"
    "$HOME/Desktop"
)
```

with optional configuration file:

```bash
~/.config/tms/config
```

### TODO

* [ ] Move defaults into configuration.
* [ ] Support environment overrides.
* [ ] Support arbitrary numbers of search paths.
* [ ] Document path depth syntax.

---

## [x] Improve session persistence format

Current format:

```text
session<TAB>window<TAB>directory
```

This is simple but limited.

### Problems

* [ ] No window index.
* [ ] No pane information.
* [ ] No layout.
* [ ] No startup commands.
* [ ] No environment metadata.
* [ ] No escaping for tabs/newlines.
* [ ] No project-level configuration.
* [ ] No versioning.

### Future format should potentially contain

```text
session
window index
window name
pane path
pane command
window layout
```

---

## [x] Version the state format

Add a format/version identifier.

For example:

```text
# tms-state-version: 2
```

### TODO

* [ ] Add version header.
* [ ] Validate version during restore.
* [ ] Support migration from old format.
* [ ] Reject unsupported versions clearly.

---

# P1 — Window & Pane Restoration

## [x] Persist window indexes

Current save format only stores:

```text
session
window name
directory
```

The original ordering/index isn't preserved reliably.

### TODO

Save:

```text
#{session_name}
#{window_index}
#{window_name}
#{pane_current_path}
```

Restore windows at their original indexes where possible.

---

## [x] Persist pane layouts

Use tmux layout information such as:

```text
#{window_layout}
```

### TODO

* [ ] Save layout.
* [ ] Restore layout.
* [ ] Handle layout restoration after panes are created.
* [ ] Handle incompatible terminal sizes gracefully.

---

## [x] Persist panes

Eventually save:

* [ ] pane index
* [ ] pane path
* [ ] pane command
* [ ] pane title
* [ ] pane dimensions
* [ ] pane layout

---

## [ ] Decide whether commands should be persisted

Possible examples:

```text
npm run dev
cargo watch
python server.py
lazygit
```

### Important

Do not blindly restart arbitrary processes during restore.

### TODO

* [ ] Distinguish between shell state and explicit startup commands.
* [ ] Allow project configuration to declare startup commands.
* [ ] Make command restoration opt-in.
* [ ] Never execute commands from untrusted project files without explicit configuration/consent.

---

# P1 — Project Configuration

## [x] Formalize `.tmux-sessionizer`

Current `hydrate()` supports:

```bash
.tmux-sessionizer
```

This is a good foundation.

Turn it into a documented project configuration mechanism.

Possible:

```text
~/workspace/project/.tms
```

or retain:

```text
~/workspace/project/.tmux-sessionizer
```

### TODO

* [ ] Decide on final filename.
* [ ] Document lifecycle.
* [ ] Document available helper functions.
* [ ] Define environment variables.
* [ ] Define security model.
* [ ] Support project-specific windows.
* [ ] Support project-specific panes.
* [ ] Support project-specific startup commands.

---

## [x] Add project-defined windows

Example conceptual configuration:

```bash
tms_window "dev" "$PROJECT_ROOT"
tms_window "git" "$PROJECT_ROOT"
tms_window "server" "$PROJECT_ROOT"
tms_window "test" "$PROJECT_ROOT"
```

### TODO

* [ ] Implement `tms_window`.
* [ ] Automatically create missing windows.
* [ ] Preserve ordering.
* [ ] Support custom directories.

---

## [x] Add project-defined panes

Example:

```bash
tms_pane "server" "npm run dev"
tms_pane "git" "lazygit"
```

### TODO

* [ ] Implement pane helper.
* [ ] Allow vertical/horizontal split.
* [ ] Allow percentage/size.
* [ ] Allow startup commands.
* [ ] Make command execution explicit.

---

## [ ] Make project configuration override defaults

Desired precedence:

```text
global defaults
    ↓
user configuration
    ↓
project configuration
    ↓
CLI arguments
```

### TODO

* [ ] Define precedence rules.
* [ ] Document them.
* [ ] Ensure behavior is predictable.

---

# P1 — CLI Improvements

## [x] Support positional project arguments

Desired UX:

```bash
tms
```

→ FZF picker

```bash
tms myproject
```

→ open/create `myproject`

```bash
tms .
```

→ open/create current directory

```bash
tms ~/workspace/myproject
```

→ open/create exact directory

### TODO

* [ ] Add positional argument support.
* [ ] Detect directories.
* [ ] Detect project names.
* [ ] Resolve ambiguous project names.
* [ ] Provide useful errors.

---

## [x] Improve `--new`

Current:

```bash
SESSION_NAME="session-$(pwd | md5sum | cut -c1-8)"
```

This is deterministic rather than random.

### Decide

Either:

### Option A — Keep deterministic behavior

Rename help text to:

```text
Create/attach to a session based on the current directory
```

### Option B — Make it genuinely random

Use something like:

```bash
session-$(openssl rand -hex 4)
```

### Recommendation

Keep deterministic behavior. It is more useful for a sessionizer.

Rename `--new` if necessary to communicate that behavior.

---

## [x] Improve `--kill`

Current:

```bash
tms --kill <session>
```

### TODO

* [ ] Validate session existence.
* [ ] Report when session doesn't exist.
* [ ] Support interactive confirmation optionally.
* [ ] Add `--force` if confirmation is introduced.

---

## [x] Rename `--remove` to `--forget`

Current:

```bash
tms --remove <session>
```

This is ambiguous.

Better:

```bash
tms --forget <session>
```

Meaning:

> Remove the session from saved state without touching the live tmux session.

### TODO

* [ ] Add `--forget`.
* [ ] Keep `--remove` as backwards-compatible alias if desired.
* [ ] Clarify help text.

---

## [ ] Add combined session operations

Potential commands:

```bash
tms --kill foo
tms --forget foo
tms --kill --forget foo
```

or:

```bash
tms destroy foo
```

### TODO

* [ ] Decide whether subcommands are worthwhile.
* [ ] Avoid unnecessary CLI complexity.

---

## [x] Improve help output

Current help is functional but minimal.

Add:

* [ ] Examples.
* [ ] Configuration location.
* [ ] State file location.
* [ ] Search path configuration.
* [ ] Project configuration.
* [ ] Explanation of save/restore behavior.
* [ ] Explanation of inside/outside tmux behavior.

Example:

```text
Examples:

  tms
      Select a project with FZF.

  tms myproject
      Open or create myproject.

  tms .
      Open the current directory.

  tms --save
      Save current tmux state.

  tms --restore
      Restore saved tmux state.
```

---

# P1 — FZF UX

## [x] Add FZF preview

Current picker only shows paths/session names.

Add preview information.

For directories:

```text
project
~/workspace/project

Git:
  branch: main
  status: clean

Recent:
  abc123 Fix authentication
```

For tmux sessions:

```text
server

dev      ~/workspace/server
git      ~/workspace/server
logs     ~/workspace/server
```

### TODO

* [ ] Add `--preview`.
* [ ] Preview git branch.
* [ ] Preview git status.
* [ ] Preview tmux windows.
* [ ] Handle non-git directories.
* [ ] Keep preview fast.

---

## [ ] Add FZF actions

Potential key bindings:

```text
Enter       Open session
Ctrl-X      Kill session
Ctrl-S      Save session
Ctrl-R      Rename
Ctrl-D      Forget saved session
```

### TODO

* [ ] Decide whether interactive actions are worth the complexity.
* [ ] Avoid making the common workflow slower.

---

## [ ] Improve FZF display

Potential display:

```text
📁 project-a
📁 project-b
📁 project-c
 tmux/server
 tmux/backend
```

### TODO

* [ ] Distinguish directories from tmux sessions visually.
* [ ] Keep colors optional.
* [ ] Don't require Nerd Fonts.
* [ ] Ensure plain terminals still work.

---

## [ ] Add search shortcuts

Potential FZF input behavior:

```text
project-name
```

searches projects.

```text
@session
```

searches tmux sessions.

### TODO

* [ ] Evaluate whether this improves UX.
* [ ] Keep implementation simple.

---

# P2 — `--prepare`

## [x] Make IDE window list configurable

Current:

```bash
windows=(dev git build test llm)
```

Move into configuration.

Potential:

```bash
TMS_WINDOWS=(dev git build test llm)
```

---

## [x] Make `--prepare` project-aware

Instead of assuming every project needs:

```text
dev
git
build
test
llm
```

allow projects to define their own layout.

Example:

```text
web project:
    editor
    server
    browser
    git

Rust project:
    editor
    cargo
    test
    git

Python project:
    editor
    server
    test
    git
```

---

## [x] Make `--prepare` idempotent

Running:

```bash
tms --prepare
```

twice should not create duplicate windows.

### TODO

* [ ] Detect existing windows.
* [ ] Rename only when appropriate.
* [ ] Create only missing windows.
* [ ] Avoid duplicate startup commands.

---

## [x] Explicitly target the current session

Don't rely on implicit tmux targets.

Get:

```bash
session="$(tmux display-message -p '#S')"
```

and explicitly target:

```bash
"$session:$window"
```

---

## [x] Make startup commands configurable

Current:

```bash
tmux send-keys -t git "git status" C-m
```

Potential configuration:

```bash
TMS_GIT_STARTUP="git status"
```

### TODO

* [ ] Make commands configurable.
* [ ] Avoid shell quoting bugs.
* [ ] Decide whether commands should run automatically or only via explicit configuration.

---

# P2 — Code Cleanup

## [x] Remove unused pane cache code

Currently present:

```bash
get_pane_id()
set_pane_id()
```

and related pane cache functionality.

### TODO

Either:

* [ ] Finish implementing pane persistence.

or:

* [ ] Delete unused code.

Do not keep abandoned infrastructure in the main script.

---

## [x] Investigate `session_cmd`

`hydrate()` contains:

```bash
if [[ ! -z $session_cmd ]]; then
```

but `session_cmd` isn't defined in the posted script.

### TODO

* [ ] Determine intended behavior.
* [ ] Reintroduce it properly if needed.
* [ ] Otherwise remove it.
* [ ] Use `${session_cmd:-}` if it remains optional.

---

## [ ] Normalize indentation

There is currently mixed indentation:

```bash
if [[ -n "$SESSION_NAME" ]]; then
	selected_name="$SESSION_NAME"
```

### TODO

* [ ] Use consistent spaces.
* [ ] Use 2 or 4 spaces consistently.
* [ ] Run `shfmt` where appropriate.

---

## [ ] Split large functions

`main()` currently handles:

* argument-selected sessions
* directory resolution
* FZF
* tmux detection
* session creation
* hydration
* switching

Break into smaller functions:

```bash
select_project()
resolve_project()
create_session()
ensure_session()
switch_session()
restore_sessions()
save_sessions()
```

---

## [x] Add constants

Potential:

```bash
TMS_CONFIG_DIR="${XDG_CONFIG_HOME:-$HOME/.config}/tms"
TMS_CONFIG_FILE="$TMS_CONFIG_DIR/config"
TMS_SESSION_FILE="$TMS_CONFIG_DIR/sessions"
```

And:

```bash
TMS_DEFAULT_SEARCH_DEPTH=1
```

---

## [x] Use `$HOME` consistently

Instead of mixing:

```bash
~/.tmux-session
```

and:

```bash
"$HOME/.tmux-sessionizer"
```

use:

```bash
"$HOME/..."
```

consistently.

---

# P2 — Error Handling

## [ ] Add explicit dependency checks

At startup:

```bash
command -v tmux >/dev/null || ...
command -v fzf >/dev/null || ...
```

### TODO

Check:

* [ ] `tmux`
* [ ] `fzf`
* [ ] `find`
* [ ] `awk`
* [ ] `mktemp`
* [ ] `md5sum` if still used

---

## [x] Provide useful dependency errors

Instead of:

```text
fzf: command not found
```

show:

```text
tms: fzf is required but was not found in PATH.
```

---

## [ ] Decide on strict shell options

Evaluate:

```bash
set -euo pipefail
```

but audit commands that intentionally return non-zero.

Especially:

* [ ] `grep`
* [ ] `tmux list-sessions`
* [ ] `tmux list-windows`
* [ ] `find`
* [ ] command substitutions

---

## [x] Add a logging/debug mode

Potential:

```bash
tms --debug
```

or:

```bash
TMS_DEBUG=1 tms
```

Output:

```text
[DEBUG] selected=/home/me/workspace/foo
[DEBUG] session=foo
[DEBUG] session exists
[DEBUG] switching to foo
```

### TODO

* [ ] Implement `log()`.
* [ ] Send debug output to stderr.
* [ ] Never pollute FZF input/output.
* [ ] Keep normal mode silent.

---

# P2 — Safety

## [x] Avoid arbitrary command execution from projects by default

`.tmux-sessionizer` is effectively executable project configuration.

### TODO

* [ ] Document this clearly.
* [ ] Consider explicit opt-in for project configuration.
* [ ] Never automatically execute untrusted repositories' configuration.
* [ ] Consider a trusted-project mechanism.

---

## [x] Protect state/config files

Use:

```bash
chmod 600
```

for files containing user-specific session information.

### TODO

* [ ] State file mode 600.
* [ ] Config file mode 600 where appropriate.
* [ ] Config directory mode 700.

---

## [ ] Quote all paths

Audit every invocation involving:

```bash
"$HOME"
"$selected"
"$dir"
"$session_name"
"$window_name"
```

### TODO

* [ ] Ensure all filesystem paths are quoted.
* [ ] Ensure tmux target names are quoted.
* [ ] Ensure generated shell commands are safely escaped.

---

# P3 — Architecture

## [x] Consider moving from flat script to subcommands

Potential future CLI:

```bash
tms
tms open
tms open myproject

tms list
tms save
tms restore

tms kill foo
tms forget foo

tms prepare
```

### TODO

Evaluate whether this improves the interface or merely adds complexity.

Don't refactor just for the sake of refactoring.

---

## [ ] Separate concerns

Ideal architecture:

```text
CLI
 │
 ├── config
 │
 ├── project discovery
 │
 ├── FZF UI
 │
 ├── tmux operations
 │
 ├── project hydration
 │
 └── state persistence
```

---

# P3 — Session Reconciliation

## [x] Define desired state vs actual state

Eventually `tms` should think in terms of:

```text
Desired:
    session: project
        window: dev
        window: git
        window: test

Actual:
    session: project
        window: dev
        window: git
```

Then:

```bash
tms restore
```

reconciles the difference.

### TODO

* [ ] Build desired-state representation.
* [ ] Inspect actual tmux state.
* [ ] Create missing sessions.
* [ ] Create missing windows.
* [ ] Create missing panes.
* [ ] Restore layouts.
* [ ] Optionally remove stale state.

---

# P3 — Better Save/Restore Model

## [ ] Save only managed sessions

Currently `--save` saves:

```bash
tmux list-windows -a
```

which means it saves everything.

Consider allowing:

```bash
tms --save
```

to save only sessions explicitly managed by `tms`.

### Possible approaches

* [ ] State marker.
* [ ] Session option.
* [ ] Naming convention.
* [ ] Configured allowlist.
* [ ] Explicit `tms` metadata.

---

## [ ] Add session metadata

Potential metadata:

```text
managed_by=tms
project=/home/me/workspace/project
created_by=tms
configuration=/home/me/workspace/project/.tms
```

Could potentially use tmux user options.

---

# P3 — Git Integration

## [ ] Improve directory discovery

For git repositories:

* [ ] Detect repository root.
* [ ] Display repository name.
* [ ] Avoid duplicate nested repository entries.
* [ ] Show current branch.
* [ ] Show dirty/clean status.

---

## [ ] Handle nested repositories

Current:

```bash
-path '*/.git' -prune
```

prevents traversal into `.git`, which is good.

But nested repositories may still produce unexpected entries.

### TODO

* [ ] Test monorepos.
* [ ] Test nested repositories.
* [ ] Test worktrees.
* [ ] Test bare repositories.

---

## [ ] Add git worktree support

Potentially treat worktrees as separate projects.

---

# P3 — Performance

## [ ] Benchmark directory discovery

Current:

```bash
find ...
```

runs every time FZF opens.

### TODO

* [ ] Measure startup time.
* [ ] Test large `$HOME/workspace`.
* [ ] Avoid expensive git operations during initial discovery.
* [ ] Consider caching if necessary.

---

## [ ] Keep FZF preview lazy

Never perform expensive git operations for every candidate simultaneously.

Only inspect the currently selected entry.

---

## [ ] Reduce subprocess spawning

Audit:

```bash
basename
tr
grep
cut
awk
pgrep
cat
wc
```

Some are perfectly fine, but several can be replaced with Bash or tmux-native operations.

Don't optimize prematurely; prioritize readability.

---

# P3 — Testing

## [ ] Create a test environment

Use temporary directories and an isolated tmux server.

### Test cases

* [ ] No tmux server.
* [ ] Existing tmux server.
* [ ] Inside tmux.
* [ ] Outside tmux.
* [ ] No sessions.
* [ ] One session.
* [ ] Multiple sessions.
* [ ] Missing project directory.
* [ ] Spaces in paths.
* [ ] Special characters in paths.
* [ ] Special characters in session names.
* [ ] Empty state file.
* [ ] Missing state file.
* [ ] Corrupt state file.
* [ ] Repeated restore.
* [ ] Repeated save.
* [ ] Repeated prepare.
* [ ] Existing windows.
* [ ] Missing windows.
* [ ] Nested git repositories.

---

## [ ] Add shellcheck

Run:

```bash
shellcheck ~/.local/bin/tms
```

Fix all meaningful warnings.

### Especially inspect

* [ ] Unquoted variables.
* [ ] SC2086.
* [ ] SC2046.
* [ ] SC2154.
* [ ] SC2164.
* [ ] SC2001.
* [ ] SC2012.

---

## [ ] Add formatting

Use:

```bash
shfmt
```

### TODO

* [ ] Choose formatting style.
* [ ] Run formatter consistently.
* [ ] Keep script readable.

---

# P4 — Documentation

## [x] Write a README

Document:

* [ ] Installation.
* [ ] Dependencies.
* [ ] Basic usage.
* [ ] FZF behavior.
* [ ] Search paths.
* [ ] Save/restore.
* [ ] Project configuration.
* [ ] `--prepare`.
* [ ] Security considerations.
* [ ] Examples.

---

## [x] Document configuration

Example:

```text
~/.config/tms/config
```

Document:

```bash
TS_SEARCH_PATHS
TS_MAX_DEPTH
TMUX_SESSION_FILE
```

and future configuration variables.

---

## [x] Document project configuration

Example:

```text
~/workspace/project/.tmux-sessionizer
```

Explain:

* [ ] When it executes.
* [ ] What environment it receives.
* [ ] What commands are available.
* [ ] How to create windows.
* [ ] How to create panes.
* [ ] Security implications.

---

# P4 — Nice-to-Have Features

## [ ] Session rename

```bash
tms --rename old new
```

---

## [ ] Session clone

```bash
tms --clone source destination
```

---

## [ ] Session export

```bash
tms --export project.tms
```

---

## [ ] Session import

```bash
tms --import project.tms
```

---

## [ ] Session cleanup

Detect dead/stale saved entries:

```bash
tms --clean
```

Example:

```text
Removing saved sessions whose directories no longer exist:
  old-project
  deleted-project
```

---

## [ ] Interactive cleanup

```bash
tms --clean --interactive
```

---

## [ ] "Open last session"

```bash
tms --last
```

---

## [ ] "Jump back"

```bash
tms -
```

Similar to shell directory navigation.

---

## [ ] Recent projects

Track recently used projects.

FZF ordering:

```text
recent project
recent project
recent project
----------------
all projects
```

---

## [ ] Favorites

Allow:

```bash
tms --favorite project
```

and show favorites at the top.

---

## [ ] Session groups

Potentially:

```text
work/
    api
    frontend
    infra

personal/
    dotfiles
    website
```

Could map to directories or explicit configuration.

---

# Desired End State

The ideal `tms` workflow should be extremely simple.

## Basic usage

```bash
tms
```

FZF appears.

Select:

```text
my-project
```

`tms`:

1. Resolves project root.
2. Finds/creates the tmux session.
3. Loads project configuration.
4. Creates missing windows.
5. Creates missing panes.
6. Restores desired layout.
7. Runs explicitly configured startup commands.
8. Switches/attaches to the session.

---

## Direct usage

```bash
tms my-project
```

No FZF required.

---

## Current directory

```bash
tms .
```

Open the current project.

---

## Save

```bash
tms --save
```

Persist managed tmux state atomically.

---

## Restore

```bash
tms --restore
```

Reconcile persisted state with the running tmux server.

Running it 1 time or 100 times should produce the same result.

---

## Prepare

```bash
tms --prepare
```

Apply the current project's configured development environment.

Should be:

* [ ] Idempotent.
* [ ] Project-aware.
* [ ] Configurable.
* [ ] Safe.

---

# Proposed Configuration Layout

Eventually:

```text
~/.config/tms/
├── config
├── sessions
└── projects/
    ├── project-a
    └── project-b
```

Project:

```text
~/workspace/project/
├── .git/
├── .tmux-sessionizer
└── ...
```

---

# Proposed Internal Model

Instead of thinking:

```text
directory -> tmux session
```

think:

```text
Project
│
├── Session
│   │
│   ├── Window
│   │   ├── Pane
│   │   ├── Pane
│   │   └── Layout
│   │
│   ├── Window
│   │   └── Pane
│   │
│   └── Window
│
└── Configuration
    ├── startup commands
    ├── windows
    ├── panes
    └── layout
```

This model will make future features significantly easier.

---

# Suggested Milestones

## Milestone 1 — Reliability

* [ ] Idempotent restore.
* [ ] Native tmux session checks.
* [ ] Safe session-name matching.
* [ ] Atomic state saving.
* [ ] Missing-state handling.
* [ ] Non-TTY handling.
* [ ] Shellcheck cleanup.

**Goal:** Make the existing implementation boringly reliable.

---

## Milestone 2 — CLI

* [ ] `tms <project>`
* [ ] `tms .`
* [ ] Better help.
* [ ] `--forget`.
* [ ] Better error messages.
* [ ] Debug mode.

**Goal:** Make common operations extremely fast.

---

## Milestone 3 — Project Configuration

* [ ] Formalize `.tmux-sessionizer`.
* [ ] Project-specific windows.
* [ ] Project-specific panes.
* [ ] Project-specific startup commands.
* [ ] Configuration precedence.

**Goal:** Make each project able to describe its own tmux environment.

---

## Milestone 4 — Persistence

* [ ] Versioned state format.
* [ ] Window indexes.
* [ ] Pane persistence.
* [ ] Layout persistence.
* [ ] Desired-vs-actual reconciliation.

**Goal:** Restore a development environment rather than merely recreating windows.

---

## Milestone 5 — FZF UX

* [ ] Preview.
* [ ] Git information.
* [ ] Session information.
* [ ] Interactive actions.
* [ ] Recent projects.

**Goal:** Make `tms` pleasant enough to use constantly.

---

## Milestone 6 — Polish

* [ ] Documentation.
* [ ] Tests.
* [ ] Performance profiling.
* [ ] Configuration cleanup.
* [ ] Remove dead code.
* [ ] Stable CLI/API.

**Goal:** Make `tms` something worth keeping for years.

---

# Priority Summary

## Must Have

* [ ] Idempotent restore
* [ ] Native tmux session detection
* [ ] Safe session matching
* [ ] Atomic save
* [ ] Missing state handling
* [ ] Proper quoting
* [ ] Shellcheck cleanup

## Should Have

* [ ] Positional project argument
* [ ] Project configuration
* [ ] Configurable `--prepare`
* [ ] Better state format
* [ ] Window index persistence
* [ ] Layout persistence
* [ ] FZF preview
* [ ] Debug mode

## Could Have

* [ ] Pane persistence
* [ ] Recent projects
* [ ] Favorites
* [ ] Session cleanup
* [ ] Import/export
* [ ] Session cloning
* [ ] Session groups

## Probably Don't Need Yet

* [ ] Complex subcommand architecture
* [ ] Heavy caching
* [ ] Database-backed state
* [ ] Network synchronization
* [ ] Over-engineered plugin system

Keep the tool small and shell-native unless real usage proves that more machinery is necessary.

---

# Final Target

**`tms` should feel like `cd` + FZF + tmux session management + project-aware development environments.**

The ideal command remains:

```bash
tms
```

Everything else should support that experience rather than get in its way.

