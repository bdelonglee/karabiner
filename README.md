# Karabiner config

Personal Karabiner-Elements configuration for macOS. Notably includes a
2-level "leader key" app launcher system: hold a leader combo, release it,
then tap a letter within a short window to launch an app (or a specific
Chrome page/profile).

## Setup

`karabiner.json` in this repo is the live config. It's symlinked from
Karabiner's expected location:

```
~/.config/karabiner/karabiner.json -> <repo>/karabiner.json
```

If you clone this repo fresh on a new machine, recreate the symlink:

```sh
mv ~/.config/karabiner/karabiner.json ~/.config/karabiner/karabiner.json.bak # if one exists
ln -s /path/to/this/repo/karabiner.json ~/.config/karabiner/karabiner.json
```

Karabiner-Elements must be installed and granted Input Monitoring /
Accessibility permissions.

### Reloading after manual edits

Karabiner normally picks up file changes automatically, but if a change
doesn't seem to apply, restart its core service:

```sh
launchctl kickstart -k gui/$(id -u)/org.pqrs.service.agent.Karabiner-Core-Service-rev2
```

### If a change "doesn't work" — check the symlink first

Karabiner-Elements saves its own config via write-to-temp-then-rename, which
replaces a symlink with a plain file instead of writing through it. This can
happen on its own (e.g. it rewrites the profile when it notices a device
change), silently breaking the link this setup depends on — after that,
edits to the repo file stop reaching the app entirely, with no error.

If a change isn't taking effect, check first:

```sh
ls -la ~/.config/karabiner/karabiner.json   # should show `-> <repo>/karabiner.json`
```

If it's a plain file instead of a symlink, recreate it (back up the plain
file first in case it has newer device-specific settings worth diffing
against, e.g. `devices` entries or `simple_modifications` added via the
Karabiner GUI):

```sh
mv ~/.config/karabiner/karabiner.json ~/.config/karabiner/karabiner.json.bak
ln -s /path/to/this/repo/karabiner.json ~/.config/karabiner/karabiner.json
launchctl kickstart -k gui/$(id -u)/org.pqrs.service.agent.Karabiner-Core-Service-rev2
```

## Simple modifications

- Caps Lock → Left Control
- Non-US Backslash → `fn` (`keyboard_fn`)

## Leader key app launcher

Each mode works the same way: hold **right ⌘** and tap the mode's letter,
release, then tap a mapped letter within 1.5s to trigger the action. If
nothing is pressed within that window, the mode disarms automatically.

Using **right Command** specifically (not left) avoids colliding with
everyday left-⌘ shortcuts like Select All.

### `right⌘ + S` — Software

| Key | App |
|-----|-----|
| m | Maya |
| p | Photoshop 2025 |
| d | DaVinci Resolve |
| c | Clip Studio Paint |
| h | Houdini FX 20.0.733 |
| n | NukeX |
| b | Blender |
| f | FileZilla |
| k | Karabiner-Elements |
| l | Lightroom Classic |
| r | RV |
| s | Spotify |
| z | Zed |

### `right⌘ + A` — Lighter apps

| Key | App |
|-----|-----|
| c | ChatGPT |
| d | Disk Expert |
| e | Google Earth Pro |
| f | FaceTime |
| g | Google Chrome |
| m | Messages |
| p | NordPass |
| r | PureRef |
| s | Spotify |
| v | NordVPN |
| w | WhatsApp |
| o | Obsidian |
| z | Zen Notes |

### `right⌘ + W` — Work (macguff.eu Chrome profile)

All Chrome actions open with `--profile-directory='Profile 4'`
(`bdelonglee@macguff.eu`).

| Key | Action |
|-----|--------|
| g | Google Chrome (blank window, work profile) |
| m | Gmail |
| a | Google Calendar |
| n | NoMachine |
| t | Todoist |
| d | Google Drive |
| c | Google Chat |
| l | LinkedIn |
| i | Discord |

### `right⌘ + P` — Personal (bdelonglee@gmail.com Chrome profile)

All Chrome actions open with `--profile-directory='Default'`
(`bdelonglee@gmail.com`).

| Key | Action |
|-----|--------|
| g | Google Chrome (blank window, personal profile) |
| m | Gmail |
| a | Google Calendar |
| t | Todoist |
| d | Google Drive |
| c | Google Chat |
| l | LinkedIn |

### `right⌘ + /` — Shortcuts mode

Same 2-level pattern as the other modes: hold, release, then tap a letter
within 1.5s to open a cheat sheet page in the browser. Pages live in
`~/.config/cheatsheet/` (centralized there, not in this repo, so pages for
other tools like tmux/aerospace can live alongside the Karabiner one).

| Key | Page |
|-----|------|
| k | Karabiner (this repo's shortcuts) |
| t | Tmux |
| a | Aerospace |

Keep `~/.config/cheatsheet/karabiner.html` in sync whenever a mapping
changes here.

## Adding more

Each mode is a rule in `complex_modifications.rules`, keyed by a
`set_variable` flag (`app_launcher_mode`, `lighter_apps_mode`, `work_mode`,
`personal_mode`, `shortcuts_mode`) that's armed by the leader manipulator and read by each letter's
`variable_if` condition. To add a letter to an existing mode, copy one of
its manipulators, change the `key_code` and the `shell_command`. To add a
new mode, copy a whole rule block with a new leader key and variable name.
