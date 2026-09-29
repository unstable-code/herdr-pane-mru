# herdr-pane-mru

**English** | [한국어](README.ko.md)

A [herdr](https://herdr.dev) plugin that makes directional pane focus (`prefix+h/j/k/l`) return to the
**most recently used pane**, like tmux.

## Why

herdr 0.9.0 picks the target of a directional move purely by geometry (`find_in_direction` in `src/layout.rs`):

```
sort key = (edge distance, larger overlap, center distance, layout order)
```

In a layout where one column is split in two, going from `B` to `C` and back ties on the first three keys,
so herdr **always lands on `A`**, the pane that comes first in layout order. It does not remember that you were in `B`.

```
┌─────┬─────┐
│  A  │     │
├─────┤  C  │     B → C → (left) ⇒ herdr: A    this plugin: B
│  B  │     │
└─────┴─────┘
```

tmux picks the most recently used pane in this case. This plugin reproduces that.

## How it works

- A `pane.focused` event hook (`bin/record`) keeps a per-tab most-recently-used (MRU) list in
  `$HERDR_PLUGIN_STATE_DIR/<tab>.mru`. It sees every focus change, whatever caused it (keys, mouse, notification jump).
- The directional actions (`bin/focus <dir>`) use **the same candidate rule as herdr** (panes on that side whose
  perpendicular axis overlaps) and insert the MRU rank into the sort key:

  ```
  herdr          = (edge distance,           overlap desc, center distance, order)
  herdr-pane-mru = (edge distance, MRU rank, overlap desc, center distance, order)
  ```

  Edge distance stays first, so history is only followed among **directly adjacent** panes. Candidates with no
  history are ordered exactly as herdr would order them.
- herdr's CLI `pane focus` only accepts a direction, so focusing a specific pane id goes through the socket API
  (`pane.focus`).
- When the pane is zoomed, or anything fails, it **hands off to herdr's built-in directional focus** so the plugin
  can never block navigation.

## Requirements

- herdr ≥ 0.9.0 (Linux / macOS)
- `bash`, `jq`, `socat`, `flock` (util-linux), found on the herdr server's `PATH`. Without `jq` or `socat` it
  falls back to the built-in move.

## Installation

```sh
herdr plugin install unstable-code/herdr-pane-mru
```

Run the same command again to update. The canonical repository is a self-hosted GitLab instance, mirrored to
[GitHub](https://github.com/unstable-code/herdr-pane-mru) because `herdr plugin install` only fetches from GitHub.

For development, link a local clone instead; the working tree is used directly, so `git pull` is the update:

```sh
git clone https://github.com/unstable-code/herdr-pane-mru.git
herdr plugin link ./herdr-pane-mru
```

To switch a linked copy to an installed one, `herdr plugin unlink unstable-code.herdr-pane-mru` first: herdr refuses
to replace a plugin that is linked from a local path (herdr 0.9.0 `ensure_replacement_allowed` in `src/cli/plugin.rs`).

In `~/.config/herdr/config.toml`, **clear** the built-in directional keys and bind the same keys to the plugin actions:

```toml
[keys]
focus_pane_left = ""
focus_pane_down = ""
focus_pane_up = ""
focus_pane_right = ""

[[keys.command]]
key = "prefix+h"
type = "plugin_action"
command = "unstable-code.herdr-pane-mru.focus-left"
description = "focus pane left (MRU)"

[[keys.command]]
key = "prefix+j"
type = "plugin_action"
command = "unstable-code.herdr-pane-mru.focus-down"
description = "focus pane down (MRU)"

[[keys.command]]
key = "prefix+k"
type = "plugin_action"
command = "unstable-code.herdr-pane-mru.focus-up"
description = "focus pane up (MRU)"

[[keys.command]]
key = "prefix+l"
type = "plugin_action"
command = "unstable-code.herdr-pane-mru.focus-right"
description = "focus pane right (MRU)"
```

⚠️ Clearing the built-in keys matters. If `focus_pane_*` is set in your config and you bind the same key in
`[[keys.command]]`, herdr treats it as a **conflict between two user bindings and disables the plugin binding**
(herdr 0.9.0 `src/config/keybinds.rs` registers built-in actions first). If `focus_pane_*` is not in your config at
all, the defaults are silently displaced and clearing them is not needed.

`description` is what herdr's help overlay (`prefix+?`) shows in the custom group. Without it all four keys show up
as `custom command` (herdr 0.9.0 `src/input/keybind_help.rs`). Matching the built-in wording (`focus pane left`)
keeps it obvious what the keys used to be.

Apply with `herdr server reload-config` (or your reload key).

## Verification

Checked on an isolated herdr 0.9.0 server (separate `HOME`) with the layout pictured above.

| Scenario | Built-in | Plugin |
|---|---|---|
| B → C → left | A | **B** |
| A → C → left | A | A |
| No pane in that direction | no move | no move |

All commands exited 0 in the plugin log; a move took about 34 ms and a record about 23 ms.

## Limitations

- Each key press runs a shell, `herdr pane layout` and a socket request, so it is a few tens of milliseconds
  slower than the built-in move.
- Every focus event spawns a `bin/record` process.
- The MRU list is per tab. A pane moved to another tab gets a new id, so its history does not carry over.

## Third-party

The candidate selection and ordering rule in `bin/focus` is modelled on herdr's
`find_in_direction` (`src/layout.rs`). herdr is licensed under Apache-2.0. No herdr
code or binary is redistributed here.

## License

[MIT](LICENSE)
