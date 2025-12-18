# VisiData configuration (OSC52 clipboard, SSH + tmux friendly)

This repository contains my VisiData configuration and a small OSC52 helper
script so copying to the local system clipboard works reliably when:

- working over SSH
- there is no display server (no X11 / Wayland, $DISPLAY unset)
- running inside tmux

This setup mirrors how OSC52 clipboard support works in tools like Neovim,
but adapted to how VisiData handles system clipboard commands.

---

## Why this is needed

VisiData’s “copy to system clipboard” commands (Y, gY, zY, gzY)
often shell out to external tools such as xclip.

On headless SSH sessions this typically fails with errors like:

    Error: Can't open display: (null)

Even when options.clipboard = 'osc52' is set, VisiData still relies on
external clipboard commands for system clipboard operations.

---

## The solution

1. Provide a custom OSC52 clipboard helper
2. Configure VisiData to use it via options.clipboard_copy_cmd
3. Ensure the OSC52 escape sequence reaches the terminal emulator

### Critical detail

VisiData runs clipboard commands with redirected stdout.
OSC52 only works if the escape sequence reaches the terminal.

The helper therefore writes directly to /dev/tty.

---

## Repository layout

This repository is assumed to be cloned into:

    ~/.config/visidata

Layout:

    ~/.config/visidata/
    ├── README.md
    ├── .visidatarc
    └── bin/
        └── osc52-copy

---

## Installation

### 1) Clone the repository

    git clone git@github.com:<youruser>/<yourrepo>.git ~/.config/visidata

---

### 2) Install the OSC52 helper on your PATH

Create a symlink into ~/.local/bin:

    mkdir -p ~/.local/bin
    ln -sf ~/.config/visidata/bin/osc52-copy ~/.local/bin/osc52-copy

Ensure the script is executable:

    chmod +x ~/.config/visidata/bin/osc52-copy

Sanity check:

    printf 'OSC52_TEST' | ~/.local/bin/osc52-copy

You should be able to paste OSC52_TEST locally.

---

### 3) Install the VisiData config

VisiData loads ~/.visidatarc, so create a symlink:

    ln -sf ~/.config/visidata/.visidatarc ~/.visidatarc

Restart VisiData if it was running.

---

## tmux configuration

OSC52 must be allowed through tmux.

Recommended minimal settings:

    set -g allow-passthrough on
    set -g set-clipboard off

Reload tmux or restart sessions after changing this.

---

## VisiData configuration files

### ~/.config/visidata/.visidatarc

    # VisiData config (Python)

    # Default CSV delimiter
    options.csv_delimiter = ';'

    # Use OSC52 helper for system clipboard copy
    import os
    options.clipboard_copy_cmd = os.path.expanduser('~/.local/bin/osc52-copy')

    # Paste-from-system-clipboard disabled by default in headless SSH
    options.clipboard_paste_cmd = ''

---

### ~/.config/visidata/bin/osc52-copy

    #!/usr/bin/env bash
    set -euo pipefail

    data="$(cat)"

    # Emit OSC52 escape sequence to the controlling terminal.
    # Writing to /dev/tty is essential when stdout is redirected.
    printf '\033]52;c;%s\007' \
      "$(printf %s "$data" | base64 | tr -d '\n')" \
      > /dev/tty

This file must be executable.

To ensure the executable bit is tracked by Git:

    git add --chmod=+x bin/osc52-copy
    git commit -m "Make osc52-copy executable"

---

## Usage (system clipboard)

In VisiData:

- Y    copy current row to system clipboard
- gY   copy selected rows
- zY   copy current cell
- gzY  copy selected rows in current column

Paste locally using your normal paste shortcut.

---

## Troubleshooting

### Nothing is copied, no error shown

Verify the helper works in the same tmux pane:

    printf 'TEST' | ~/.local/bin/osc52-copy

If this does not copy, the issue is ou

