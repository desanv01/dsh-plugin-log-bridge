# dsh-plugin-log-bridge

**A DeepSeek Harness plugin that writes the harness's own runtime warnings and errors to stderr, so a boot that fails after startup can say why.**

![MIT](https://img.shields.io/badge/license-MIT-green)
![node](https://img.shields.io/badge/node-%E2%89%A522-339933)
![dsh](https://img.shields.io/badge/dsh-0.1.x%20%7C%200.2.x-1f6feb)
![npm](https://img.shields.io/npm/v/dsh-plugin-log-bridge)
![status](https://img.shields.io/badge/status-working%2C%20new-brightgreen)

The harness keeps its log records in an in-memory ring and exports them nowhere. An error that happens after
startup — a session that cannot activate, a preset that fails to mount — leaves no trace once the process is
gone, so whatever supervises the process can only report *that* the boot failed, not *why*. Warnings are worse:
the default exporter level filters them out of the ring entirely.

This plugin attaches an exporter that writes every `error` and `warn` to **stderr**, which any supervisor
already captures, and records session-activation failures the same way.

> **Project status:** 1.0.0. Verified against the DeepSeek Harness **0.1.x** composition it was written for,
> where it was exercised in the desktop app that bundles it. **Not yet verified inside the official DeepSeek
> Harness desktop app (0.2.0-rc.2)** — that verification is pending. The plugin has no UI and requires no other
> package, so the surface it touches is small.

## Why this exists

A harness that fails to start is the worst possible time to have no diagnostics. When it fails *after* the
ready line — when a session cannot activate, or a plugin row throws — the process exits and takes its log ring
with it. The only record is whatever the parent process chose to capture.

Node's own stderr survives the process, which makes it the one durable channel available without adding a logger
dependency. This plugin uses nothing else: it imports nothing at module scope, so there is nothing for the
loader to resolve and nothing that can fail to install.

## What it does

- **Bridges `error` and `warn` records** to stderr, one prefixed line per record, with stack traces indented
  under their first line.
- **Replays the buffered errors** that were logged before the plugin loaded. Warnings from that window are
  already gone — the default level filtered them out of the ring before this plugin existed — and the README
  says so rather than pretending otherwise.
- **Records session-activation failures** from `api-session/error`, which never reach the logger at all.
- **Rate-limits itself**: 100 messages per 60-second window, then a single line saying how many were dropped.
  An error in a loop cannot flood the log.
- **Writes a ready line once per launch**, so a log without it can be read as predating the bridge.
- **Cannot break the boot.** Every attachment step is wrapped. A bridge that can take down the startup it
  exists to explain would be worse than no bridge.

## How it works

`apply(ctx)` does three things, each in its own `try`:

1. If `ctx.logger.buffer` is an array, it replays the buffered `error` records.
2. It attaches an exporter with `levels: { default: 2 }` (Cordis levels: error 0, info 1, warn 2) to
   `ctx.logger`.
3. It subscribes to `api-session/error` for session-activation failures.

Every write goes through one rate-limited writer. Each line carries the prefix `[harness-log]`, and that prefix
matters: a loader that inspects stderr to decide which plugin broke a boot can skip prefixed lines, so bridged
runtime chatter can never change which plugin a recovery pass blames.

The module exports its formatting and rate-limiting helpers (`formatMessage`, `bridgeLines`,
`createLimitedWriter`) as pure functions with an injectable clock, so they can be exercised without a harness.

## Requirements

- Node.js 22 or later (the harness ships its own; you do not need a separate install).
- A DeepSeek Harness profile that can load a plugin bundle — 0.1.x or 0.2.x.
- A parent process that captures the child's stderr. Running `dsh` in a terminal satisfies this.

## Quick start

### Install from npm

```powershell
dsh plugin --profile web add dsh-plugin-log-bridge
```

### Install into the desktop app

Point the app at this folder with `--add-plugin <path>`, or use the official app's plugin manager to install
the npm package. Installing adds the bundle row; the exported lines then appear on stderr beside the harness's
own output.

### Build from source

There is nothing to build. The package is two source files and a patch:

```powershell
git clone https://github.com/desanv01/dsh-plugin-log-bridge.git
cd dsh-plugin-log-bridge
dsh plugin --profile web add .
```

### Smoke test

The pure helpers are testable with no harness at all:

```powershell
node --input-type=module -e "import('./index.js').then(m => {
  console.log(m.formatMessage({ args: ['hello', new Error('boom')] }));
  console.log(m.bridgeLines('error', 'harness', 'line one\nline two').join('|'));
})"
```

Expect a formatted message and two prefixed lines, the second indented.

## Configuration

There is none. The bridge has no `Config` schema and takes no options, by design — a diagnostic that needs
configuring is not the one you want when something is broken at boot.

The constants that could reasonably be tuned live at the top of `index.js` and are deliberately not exposed:

| Constant | Value | Meaning |
| --- | --- | --- |
| `EXPORT_LEVEL` | `2` | Admit up to `warn` (error 0, info 1, warn 2) |
| `WINDOW_MS` | `60_000` | Rate-limit window |
| `WINDOW_LIMIT` | `100` | Lines per window before dropping |

## Slots used

None. This is a host-only plugin; it contributes no UI and registers into no slot.

## Where things live

| Piece | Path |
| --- | --- |
| Host half | `index.js` |
| Bundle patch | `cordis.patch.yml` — inserts the row `desktop-log-bridge` |
| Bridged lines | on the harness's **stderr**, each prefixed `[harness-log]` |
| Ready line | on **stdout**, once per launch |

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| No `[harness-log]` lines at all | Nothing has logged an error or warning yet | Force one with a deliberately broken plugin, or check that the row is in the profile |
| No ready line | The plugin did not load | Check the profile's `cordis.patch.yml` for the `desktop-log-bridge` row, or that the bundle was added |
| Lines are truncated | The supervisor buffers stderr | Read the supervisor's log, not the terminal |
| A `dropped N message(s)` line | The rate limit was hit | Expected under a loop; the count is reported so it is not silent |
| Warnings from before load are missing | The default exporter level filtered them out of the ring | Not recoverable — the ring never had them |

## Current status

Working, and small enough to audit in one sitting: 144 lines of host code, no dependencies, no UI.

Verified: formatting and rate-limiting as pure functions; the buffered-error replay; session-error bridging;
and that a failing attachment path leaves the boot intact. Exercised in the desktop app that first bundled it.

Not yet verified: the plugin running inside the **official** DeepSeek Harness desktop app (0.2.0-rc.2). The
host surface it uses — `ctx.logger`, `ctx.on('api-session/error')` — is part of the documented composition, not
private API, so the port is expected to be uneventful. That expectation is not evidence.

## Roadmap

- [x] Export `error` and `warn` to stderr with a stable prefix
- [x] Replay errors buffered before load
- [x] Record session-activation failures
- [x] Rate-limit with an explicit dropped-count line
- [x] Never break the boot it explains
- [ ] Verify in the official desktop app (0.2.0-rc.2)
- [ ] Publish to npm (the package is not on the registry yet)

## Security

This plugin writes harness log output to **stderr**. That output can contain prompt text, file paths, tool
arguments and error messages — anything the harness or a plugin chose to log as an error or a warning. It does
not redact. Treat captured stderr with the same care as the session logs themselves, and do not paste it into
a public issue without reading it first.

It makes no network calls, reads no credentials and writes no files.

Report a vulnerability through the process in [SECURITY.md](SECURITY.md).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). The plugin is small and the review bar is: does it stay out of the
boot path, and does it stay honest about what it can and cannot see?

## License and attribution

MIT — see [LICENSE](LICENSE).

Extracted from `desanv01/deepseek-harness-desktop-app`, where it shipped as the bundled log bridge.
