# Kitty provider notes

This fork is for local use and is not intended to be pushed back to origin. Keep the notes here so future syncs from upstream can preserve the local Kitty-provider rationale and test checklist.

## Goal

Add Kitty as another terminal surface provider for interactive subagents, alongside cmux, tmux, zellij, and WezTerm.

Kitty support uses Kitty remote control (`kitty @ ...`) and treats a Kitty window as the subagent surface id.

## Local-only fork policy

- Do not push this branch/fork to the upstream origin.
- Periodically sync from upstream and resolve conflicts locally.
- Keep local changes small and isolated around the mux provider layer where possible.
- If upstream renames or splits `pi-extension/subagents/cmux.ts`, reapply the Kitty backend as the same surface operations: create, send, read, escape, close, and title.

## Requirements

Kitty must allow remote control from the Pi process:

- set `allow_remote_control yes` in `kitty.conf`
- recommended: also set `listen_on unix:/tmp/kitty-$UID.sock` so `KITTY_LISTEN_ON` is available to Pi

The listen socket avoids Kitty remote-control replies travelling through Pi's terminal pty, which can interfere with the TUI input stream during screen polling.

A quick manual check:

```bash
kitty @ ls --self
```

If this fails, `PI_SUBAGENT_MUX=kitty` should be treated as unavailable and Pi will show the setup hint.

## Surface mapping

| Subagent operation | Kitty remote-control command |
| --- | --- |
| create surface | `kitty @ launch --type=window --dont-take-focus --cwd <cwd> --title <name>` |
| send command | `kitty @ send-text --match id:<window-id> --stdin` |
| read screen | `kitty @ --to "$KITTY_LISTEN_ON" get-text --match id:<window-id> --extent screen` |
| send Escape | `kitty @ send-key --match id:<window-id> escape` |
| close surface | `kitty @ close-window --match id:<window-id> --ignore-no-match` |
| current tab title | `kitty @ set-tab-title ...` |
| window/workspace title | `kitty @ set-window-title ...` |

## Test checklist

Run unit tests:

```bash
npm test
```

Run mux integration tests inside Kitty:

```bash
PI_SUBAGENT_MUX=kitty npm run test:integration
```

For a cheaper smoke test, run only the mux surface test if needed:

```bash
PI_SUBAGENT_MUX=kitty node --test --test-concurrency=1 test/integration/mux-surface.test.ts
```

## Known caveats

- The focus-preservation integration test is skipped for Kitty for now; Kitty launch uses `--dont-take-focus`, but the existing focus helper only understands cmux/tmux.
- Kitty support creates subagent windows in the current tab. If a future workflow wants tabs or OS windows, add an explicit option rather than changing the default silently.
- Screen reads intentionally require `KITTY_LISTEN_ON`. Pi-backed subagents also write file sentinels/sidecars so normal completion does not depend on Kitty screen polling.
- Nested subagent spawning from a Kitty child depends on remote-control environment/config being available to that child process.
