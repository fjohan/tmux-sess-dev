# tmux-sess

To import the inspected older prototype format, run `./migrate-prototype`.
It reads `~/.local/state/tmux-sess-proto` and installs into the current state
directory without changing the originals or overwriting existing snapshots.
Optional arguments select source and destination directories. Blank prototype
names are inferred from directory names with their trailing numeric IDs removed.
Missing shell histories become empty; Codex auto-resume is disabled. Layouts,
directories, output and pane selection are retained. Hibernation times use the
prototype completion-file modification times. Each import includes a
`migration-notes` file documenting unavailable metadata.

Hibernate an individual tmux session, closing its panes and removing it from
`tmux list-sessions`. Wake recreates the session detached.

```bash
./tmux-sess new my-session
./tmux-sess hibe my-session
./tmux-sess wake my-session
./tmux-sess list
tmux attach-session -t '=my-session'
```

`new NAME` creates `~/data/dev/NAME` if needed, opens one tmux window with two
stacked Bash panes in that directory, and starts `codex` in the upper pane. It
attaches from a terminal, or switches clients when run inside tmux. Without a
terminal it leaves the session detached. An existing live session is attached;
an existing snapshot is woken and attached. Project files are never removed.
Names for new projects must not contain slashes, dots, colons or newlines.
Set a different project parent once per machine:

```bash
./tmux-sess wd ~/projects
./tmux-sess wd                  # show the effective directory
./tmux-sess new my-project      # creates ~/projects/my-project
```

The directory must already exist. Relative paths and paths with spaces work;
the setting is stored as an absolute path in the state directory. It affects
future new projects; existing sessions and snapshots keep their directories.
`TMUX_SESS_DEV_HOME`, when set, overrides the saved setting. Without either,
the default is `~/data/dev`. Completion also offers directories for `wd`.

`list` prints an aligned table with session name, status (`hibernated`, `running`,
or `not saved`), Codex status, and last hibernation time:

```text
SESSION     STATUS      CODEX    LAST HIBERNATED
my-project  running     yes      2026-10-23T14:30:00+02:00
another     hibernated  resume   2026-10-22T18:10:00+02:00
```

CODEX is `yes` for a detected running Codex process, `resume` for a hibernated
conversation saved for wake, `no` when none is detected, or `unknown` when a live
process cannot be inspected. Listing uses one tmux request for all live panes
and one process-table scan, rather than querying each saved session separately.
`list --tsv` returns four tab-separated fields without a heading for scripts;
`list --names` returns just known names without contacting tmux.

The timestamp uses
ISO 8601 with the local UTC offset, for example `2026-10-23T14:30:00+02:00`.
It remains visible after waking and updates on the next hibernation. `-` means
the session has never been hibernated or its snapshot predates timestamp support.
`new` registers sessions immediately. `not saved`
means a registered session disappeared before it was hibernated. Ordinary tmux
sessions without snapshots or registration are excluded. With no known sessions,
`list` prints a short hint; `--tsv` and `--names` produce no output.

Enable Bash completion in the current shell:

```bash
eval "$(./tmux-sess completion bash)"
```

Add that line (using the script's absolute path) to `~/.bashrc` for future shells.
Tab completes commands, flags, and names with spaces. `hibe` offers live sessions,
including ordinary tmux sessions that can be saved; `wake` offers hibernated
snapshots; `new` offers known and live names. Prefix matching takes priority,
with substring matching as a fallback. Use `./tmux-sess -h` or `--help` for
command descriptions, examples, settings and restore limitations.

Requires Bash 4.4+, tmux (written against 3.4), and standard Linux utilities
including `flock` and `sha256sum`. `tmux-sess` is a standalone Bash script. Run from another terminal or session:
hibernating the session containing this command would also terminate the saver.

Before hibernating, leave **shell panes at empty Bash prompts**, with all
foreground programs other than Codex and background/stopped jobs finished. Run
`hibe` from another session or terminal. Codex panes must have been launched from
an interactive Bash shell, rather than replacing it with `exec codex`.

The saver first validates all panes, then sends Ctrl-C to Codex (at most twice),
waits for it to exit to Bash, and records the UUID from its printed
`codex resume UUID` command. An active Codex turn may be interrupted. The ID comes
from output logged during this exit, never from `resume --last` or another
conversation. If exit or ID capture fails, hibernation stops with the tmux
session alive. Codex may already have closed; consult its exit message. This
relies on the CLI's exit footer. Detection checks the foreground process name
and, on Linux, the foreground process tree's executable paths (including the
npm Node launcher), so renamed process titles are supported. Unsupported wrappers
or changed footer formatting fail conservatively.

On wake, Codex panes restore shell history and run `codex resume UUID`. Their
saved screen remains in the snapshot's pane `screen` file but is not replayed,
so Codex can redraw its conversation without duplicate output. This also means
outside-Codex shell output in those panes is archived rather than displayed.
Ordinary panes restore their screen and shell history as before. Codex's own
saved conversation must still exist; the tmux snapshot does not back it up or
save CLI flags, model overrides, or remote-server connection settings.

The saver sends a history export command to each Bash prompt. Do not type in
those panes while saving. It refuses other foreground programs, jobs it can
detect, and windows linked into other sessions. Prompt detection cannot reliably distinguish a shell prompt
from Bash's `read` builtin or an unfinished command: you must check this yourself.

Saved state includes window names and indices, pane layout and sizes, selected
window/pane, zoom, working directories, per-pane Bash history, and the screen
plus available tmux scrollback (including text colors). Saving completes before
`kill-session` runs. Other sessions are left running.

This recreates shells; it is not process checkpointing. Other running applications,
shell variables/functions defined interactively, virtualenv activation, shell
options, process IDs, and terminal cursor state are not restored. Bash startup
files run again. Scrollback is replayed as text, followed by a fresh prompt;
expect an extra prompt and possible wrapping differences. Output tmux has already
discarded cannot be recovered. Set tmux's `history-limit` high enough *before*
waking to retain all replayed output. Custom `.bashrc` history or prompt hooks may
change the restored history. Session-specific tmux options, bindings, hooks, and
client attachments are not saved.

Snapshots live under `${XDG_STATE_HOME:-~/.local/state}/tmux-sess`, or the directory
specified by `TMUX_SESS_HOME`. Each session uses a SHA-256 directory name and a
`name` file containing its original name. Files are created privately and may
contain sensitive output and commands. The state directory must be trusted.

Wake retains the snapshot for recovery. Hibernating again replaces it only after
the new save completes. A failed save keeps the previous snapshot and leaves the
live session alive, although history export commands may already have run in
some panes. A failed wake removes its partial session and keeps the saved state.

For an isolated integration check, run `bash tests/integration.sh`. It starts
its own tmux server and does not touch your default server.
`bash tests/codex-integration.sh` tests the Codex lifecycle using a local fake
CLI (requires gcc; no API calls). `bash tests/dispatch.sh` checks new-session
dispatch with a mock tmux and works without socket access.
`bash tests/process-detection.sh` checks renamed Codex process detection on Linux
(requires gcc; no tmux socket or API calls).
`bash tests/list-completion.sh` tests list formatting, batched queries, Codex
status, help and completion without a real tmux server.
