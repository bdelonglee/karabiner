# Karabiner project — status notes for next session

## What this repo is

A 2-level "leader key" app launcher built on Karabiner-Elements complex
modifications, plus the user's actual `karabiner.json`.

- `karabiner.json` here is the live config.
- `~/.config/karabiner/karabiner.json` is a **symlink** pointing to it, so
  editing the file in this repo *is* editing the live config — no copy step
  needed.
- `README.md` documents every mode and letter mapping (keep it in sync with
  `karabiner.json` when adding/changing letters).
- `cheatsheet.html` is a standalone styled page mirroring the README's
  tables, opened via a dedicated shortcut (see below).

## What's been built so far

Three independent leader modes, all triggered with **right ⌘** (not left,
to avoid clashing with everyday left-⌘ shortcuts like Select All) + a mode
letter, then a follow-up letter within 1.5s:

- `right⌘ + S` → software/creative apps (Maya, Photoshop, Houdini, Nuke,
  etc.)
- `right⌘ + A` → lighter apps (ChatGPT, Spotify, WhatsApp, etc.)
- `right⌘ + W` → work apps, all Chrome-based actions using
  `--profile-directory='Profile 4'` (the `bdelonglee@macguff.eu` Chrome
  profile — profile dirs are mapped to emails via
  `~/Library/Application Support/Google/Chrome/Local State` →
  `profile.info_cache`)

Each mode is implemented as one `complex_modifications` rule: the leader
manipulator does `set_variable` to arm a mode flag (e.g.
`app_launcher_mode`), with a `to_delayed_action.to_if_invoked` clearing it
after 1500ms of inactivity. Each letter manipulator is conditioned on
`variable_if` that flag == 1, runs its `shell_command` (`open -a '...'`),
then resets the flag to 0.

**Important bug already fixed once:** do NOT add `to_delayed_action.to_if_canceled`
to the leader manipulator. `to_if_canceled` fires immediately on ANY
subsequent keypress — including the intended follow-up letter — and races
the letter-manipulator's own condition check, silently defeating the whole
mode. Only `to_if_invoked` (pure timeout, nothing else pressed) should be
used.

Additionally there's a standalone (non-leader, single combo) shortcut:
`right⌘ + /` → `open cheatsheet.html`, meant to pull up the shortcut
reference on demand.

## Current unresolved issue

`right⌘ + /` behaves **inconsistently**: sometimes it plays a system sound
(`afplay /System/Library/Sounds/Glass.aiff`) that was only ever used
*temporarily* as a debugging probe and has since been reverted out of
`karabiner.json` — yet it still sometimes fires. Other times the same combo
just types a literal `/` (unmatched passthrough), and rarely it correctly
opens `cheatsheet.html`. Behavior also seemed to differ depending on which
app was frontmost, which shouldn't happen since the rule has no
frontmost-application condition.

### Diagnosis

This points to **two (or more) Karabiner background processes racing for
the same keyboard grab** — one has picked up the current, correct
`karabiner.json`, the other is stale and still has the old in-memory rule
set (from when the file briefly contained the `afplay` debug action).
Whichever process happens to be holding the grab at a given moment decides
what happens to the keystroke, hence the randomness.

What was tried and ruled out:
- Config file content itself was verified correct on disk multiple times
  (`python3 -m json.tool` + `karabiner_cli --lint-complex-modifications`
  both pass, and manual dump of parsed manipulators showed only one
  correct `slash` manipulator, no leftover `afplay`).
- Restarting the per-user agent repeatedly did change its PID each time
  (`launchctl kickstart -k gui/$(id -u)/org.pqrs.service.agent.Karabiner-Core-Service-rev2`),
  confirmed via before/after `ps aux`, so that specific process was
  definitely cycling and reading the latest file.
- There is **also** a root-owned `Karabiner-Core-Service` process
  (visible in `ps aux` as `root ... Karabiner-Core-Service.app`) that has
  been running continuously since before this session started and was
  never restarted.
- Newer Karabiner-Elements versions register their daemons via macOS's
  `SMAppService` API rather than classic plist files — there is **no**
  `/Library/LaunchDaemons/*pqrs*` file to unload/reload directly, which is
  why the usual `launchctl unload/load <plist>` approach doesn't apply
  here.
- `sudo` is not passwordless on this machine (`sudo -n true` fails), so a
  root-level service restart couldn't be done without the user typing
  their password — intentionally left for the user rather than scripted.

### Next step (planned when this session left off)

The user is about to **reboot the Mac**, which cleanly kills every
Karabiner process (root and user-level) and restarts from scratch reading
the current, correct `karabiner.json`. This should resolve the randomness
entirely, since there will only be one process instance again.

### How to verify after reboot

1. Press `right⌘ + /` several times in a row. Expect: cheat sheet opens in
   Chrome every time, no sound, no literal `/` typed.
2. Sanity check only one Core-Service process is running:
   ```sh
   ps aux | grep "Karabiner-Core-Service" | grep -v grep
   ```
   Should show exactly one root process and (if the Settings app / agent
   is active) one user process — not duplicates of the same UID.
3. If the issue somehow persists after reboot, the next thing to try is
   an explicit reinstall/repair via Karabiner's bundled
   `/Library/Application Support/org.pqrs/Karabiner-Elements/repair.sh`
   (not yet used — inspect it before running, it's Karabiner's own
   maintenance script, not something we wrote).

### Standing reload habit for future edits

For any future manual edit to `karabiner.json` in this repo, the routine
that's been working (aside from the current bug) is:

```sh
python3 -m json.tool ~/.config/karabiner/karabiner.json > /dev/null && echo "VALID JSON"
launchctl kickstart -k gui/$(id -u)/org.pqrs.service.agent.Karabiner-Core-Service-rev2
```

Optionally lint just the `rules` array before reloading (wrap it as
`{"title": "...", "rules": [...]}` and run
`karabiner_cli --lint-complex-modifications <file>`), since the file's
overall JSON validity doesn't guarantee Karabiner's own schema accepts
every manipulator.
