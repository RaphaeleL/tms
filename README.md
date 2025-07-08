# TMS 

> Inspired by [ThePrimeagen/tmux-sessionizer/](https://github.com/ThePrimeagen/tmux-sessionizer/)

A powerful Bash utility to create, attach, or switch between `tmux` sessions based on a directory or predefined session name. It enhances developer workflows by combining `tmux` with fuzzy directory search.

## Features

* Fuzzy finder (`fzf`) to select a directory or existing `tmux` session.
* Create or attach to `tmux` sessions named after directory basenames.
* Optional hydration via `.tmux-sessionizer` script inside project directories.
* Supports custom session name creation via `-c` flag.
* Caches pane IDs for future enhancements (not yet actively used).

## Requirements

* [`tmux`](https://github.com/tmux/tmux)
* [`fzf`](https://github.com/junegunn/fzf)
* Bash

## Installation

Place the script somewhere in your `$PATH`, e.g., `~/.local/bin/tms`, and make it executable:

```bash
chmod +x ~/.local/bin/tms
```

## Usage

```bash
tms # will run fzf to filter tmux sessions 
tms -c <session_name> # will use <session_name>
```

### Options

* `-c <session_name>`, `--create <session_name>`
  Create or switch to a `tmux` session with the given name. Tries to find a matching directory in search paths.

* `-h`, `--help`
  Show help and usage information.

If no arguments are passed, a fuzzy picker (`fzf`) will open to select from:

* Directories within defined search paths
* Existing `tmux` sessions (excluding the current one, if inside tmux)

## Configuration

The script searches for directories inside the following paths (in order):

```bash
$HOME/Projects/
$HOME/.config
$HOME/dev/fco
$HOME/dev/fco/develop
$HOME/dev/fco/
$HOME/dev/code
$HOME/dev/phd
```

These can be customized by editing the `TS_SEARCH_PATHS` variable inside the script.

## Author

This script was designed to streamline `tmux` session workflows based on directory structures and session names.
