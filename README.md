# tmux-sess

Hibernate individual tmux sessions and restore them later, including pane layouts,
working directories, shell history, and scrollback. Codex conversations can be
closed when hibernating and resumed when waking.

`tmux-sess` is a standalone Bash script. Hibernated sessions disappear from tmux;
waking creates fresh shells with the saved state.

## Requirements

- Linux, Bash 4.4+, and tmux (developed against tmux 3.4).
- Standard utilities including `flock`, `sha256sum`, `readlink`, `ps`, and `date`.
- Codex CLI on your `PATH` for creating Codex workspaces and resuming conversations.

## Quick start

Run from the repository directory:

```bash
chmod +x tmux-sess

# Choose an existing directory for your projects (default: ~/data/dev).
./tmux-sess wd ~/projects

# Create ~/projects/my-project, open two stacked panes, and start Codex above.
./tmux-sess new my-project
```

From **another terminal or tmux session**:

```bash
./tmux-sess hibe my-project   # save and close the session
./tmux-sess list              # inspect saved sessions
./tmux-sess new my-project    # wake and attach
```

You can also hibernate an existing tmux session that was created manually.

## Commands

| Command | Behavior |
| --- | --- |
| `new NAME` | Create a two-pane Codex workspace, wake a saved session, or attach to a running session. |
| `hibe NAME` | Save and close one session, replacing its previous snapshot. |
| `wake NAME` | Restore a saved session detached. |
| `list` | Show known sessions, Codex status, and last hibernation time. |
| `list --tsv` | Print four tab-separated fields without a heading. |
| `list --names` | Print known session names only. |
| `wd DIRECTORY` | Save the parent directory for new projects. |
| `wd` | Show the effective project directory. |
| `completion bash` | Print Bash completion setup. |
| `-h`, `--help` | Show detailed help. |

`new` attaches from a terminal, switches clients when run inside tmux, and leaves
the session detached when no terminal is available. It never removes project
files. New project names cannot contain slashes, dots, colons, or newlines.

To attach after `wake`:

```bash
tmux attach-session -t '=my-project'
```

### Session list

```text
SESSION     STATUS      CODEX    LAST HIBERNATED
my-project  running     yes      2026-10-23T14:30:00+02:00
another     hibernated  resume   2026-10-22T18:10:00+02:00
```

- **Status:** `running`, `hibernated`, or `not saved` if a registered session
  disappeared before being hibernated.
- **Codex:** `yes` when running, `resume` when saved for wake, `no` when none is
  detected, or `unknown` when a live process cannot be inspected.
- **Last hibernated:** an ISO 8601 timestamp with its UTC offset, or `-` when
  unavailable. Waking keeps this timestamp; hibernating again updates it.

The list includes sessions registered by `new` or saved with `hibe`. Ordinary
unregistered tmux sessions are excluded. Listing batches live pane information
into one tmux query and one process-table scan.

### Bash completion

Enable completion in the current shell:

```bash
eval "$(./tmux-sess completion bash)"
```

For future shells, add the same command to `~/.bashrc` using the script's absolute
path. Completion supports commands, flags, directory paths, and session names
with spaces. It offers live sessions for `hibe`, hibernated sessions for `wake`,
and both for `new`. Prefix matches take priority, with substring matching as a
fallback.

## What gets restored

| State | Restored? |
| --- | --- |
| Window names and indices | Yes |
| Pane layout, sizes, selection, and zoom | Yes |
| Working directory in each pane | Yes |
| Per-pane Bash history | Yes |
| Screen and available scrollback, including text colors | Replayed in ordinary shell panes |
| Codex conversation | Resumed using its saved conversation ID |
| Running programs and in-memory shell state | No |

For Codex panes, shell history is restored and `codex resume UUID` redraws the
conversation. The old screen is archived in the snapshot but not replayed,
including any output from commands previously run outside Codex in that pane.

## Before hibernating

- Run `hibe` from another terminal or session.
- Leave shell panes at **empty Bash prompts**, with background and stopped jobs
  finished. Do not type into the panes while saving.
- Launch Codex from interactive Bash so closing it returns to the shell;
  avoid `exec codex`.

Hibernation validates panes, closes Codex with Ctrl-C (at most twice), and records
its resume ID from the exit message. An active Codex turn may be interrupted.
If Codex does not exit or its ID cannot be captured, hibernation stops and leaves
the tmux session alive, although Codex may already have closed. Unsupported
foreground programs and windows shared with other sessions are refused.

Prompt detection cannot distinguish every Bash builtin or unfinished command
from an idle prompt. Check that shell panes are ready before saving.

## Configuration and storage

```bash
./tmux-sess wd ~/projects
./tmux-sess wd
```

The directory must already exist. Relative paths and paths with spaces are
supported; the setting is saved as an absolute path. It affects new projects,
while existing sessions and snapshots keep their directories.

| Setting | Purpose | Default |
| --- | --- | --- |
| Saved `wd` | Parent directory for new projects | `~/data/dev` |
| `TMUX_SESS_DEV_HOME` | Override the saved project directory | Unset |
| `TMUX_SESS_HOME` | Override the snapshot and settings directory | `${XDG_STATE_HOME:-~/.local/state}/tmux-sess` |

Snapshots use private permissions and can contain sensitive output and command
history. Keep the state directory trusted. Waking retains the snapshot for
recovery; the next successful hibernation replaces it. A failed save preserves
the previous snapshot and live session. A failed wake removes its partial
session and retains the snapshot.

## Limitations

This restores a workspace rather than checkpointing processes. Interactive shell
variables and functions, virtual environments, shell options, process IDs, and
exact cursor state are not preserved. Bash startup files run again and custom
history or prompt hooks may affect restoration.

Replayed output may wrap differently and ends with a fresh prompt. Output tmux
has already discarded cannot be recovered. Set tmux's `history-limit` high enough
before waking to retain the replayed scrollback. Session-specific tmux options,
bindings, hooks, and attached clients are not saved.

Codex restoration depends on its exit-message format and its own saved
conversation remaining available. Snapshots do not back up Codex conversations
or preserve CLI flags, model overrides, or remote-server settings.

## Tests

Tests that work without a tmux socket:

```bash
bash tests/dispatch.sh
bash tests/list-completion.sh
bash tests/process-detection.sh  # requires gcc
```

Integration tests start an isolated tmux server:

```bash
bash tests/integration.sh
bash tests/codex-integration.sh # requires gcc; uses a fake CLI, no API calls
```
