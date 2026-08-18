# tms

`tms` is a small Bash tmux session manager: use it to find a project, create or
switch to its session, and persist a window/pane layout without restoring
arbitrary running commands.

## Requirements

- Bash 4+
- [tmux](https://github.com/tmux/tmux)
- [fzf](https://github.com/junegunn/fzf) only for the interactive picker
- `find`, `awk` and `mktemp`

## Installation

Place both `tms` and `tms-core` in the same directory in `$PATH`:

```bash
chmod +x tms tms-core
install -d "$HOME/.local/bin"
install -m 755 tms tms-core "$HOME/.local/bin"
```

## Usage

```bash
tms                       # choose a project or live session with FZF
tms my-project            # resolve a project name in configured search paths
tms .                     # create/switch to the current project's session
tms ~/code/project        # create/switch to an exact directory
tms --create scratch      # attach/create a named session in the current directory
tms --save                # atomically save current tmux pane/window state
tms --restore             # idempotently reconcile saved state
tms --prepare             # create missing configured windows in the current session
```

Run `tms --help` for the complete command reference. `--forget NAME` removes
saved state only; `--remove` remains a compatibility alias. `--kill NAME` asks
for confirmation when stdin is interactive; pass `--force` to skip it.

## Configuration

The optional user configuration is shell code at
`$XDG_CONFIG_HOME/tms/config` (default: `$HOME/.config/tms/config`). It is
loaded before CLI arguments, so CLI arguments have final precedence.

```bash
# ~/.config/tms/config
TS_SEARCH_PATHS=("$HOME/workspace:2" "$HOME/Desktop:1")
TMS_WINDOWS=(editor server test git)
TMS_GIT_STARTUP='git status' # only used by --prepare
```

A search path may end in `:depth`; otherwise `TMS_DEFAULT_SEARCH_DEPTH` (1) is
used. Set `TMUX_SESSION_FILE` to override the state-file location. The state
file defaults to `$XDG_CONFIG_HOME/tms/sessions` and has a version header.

### Project configuration

A project may define `.tmux-sessionizer`. It is shell code and therefore must
be treated as untrusted repository content. It is **not loaded by default**.
Opt in per invocation with `TMS_TRUST_PROJECT_CONFIG=1 tms .`.

The file receives `PROJECT_ROOT` and `TMS_SESSION`, and may call:

```bash
# .tmux-sessionizer
tms_window "editor" "$PROJECT_ROOT"
tms_window "server" "$PROJECT_ROOT"
tms_pane "server" "npm run dev" vertical 50
```

`tms_window` creates a missing named window. `tms_pane` creates a pane in an
existing window and executes its explicitly declared command. Project config
is applied when a session is first created and with `--prepare`; it is not run
while restoring saved state.

## Persistence and safety

`--save` writes a temporary file, validates it, and atomically replaces the
state file. It records pane paths, indexes, and layouts, but never saves or
restarts arbitrary pane commands. `--restore` creates only missing resources,
so repeated restores do not duplicate windows or panes. Missing state files,
missing tmux servers, and missing saved directories are handled gracefully.

Use `TMS_DEBUG=1 tms …` (or `tms --debug …`) for diagnostics on stderr.
