# Changelog

All notable changes to this project are documented here. The format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

- Verify the plugin inside the official DeepSeek Harness desktop app (0.2.0-rc.2).
- Publish to npm.

## [1.0.0]

First release as a standalone package. Extracted unchanged from
`desanv01/deepseek-harness-desktop-app`, where it shipped as the bundled log bridge.

### Added

- An exporter that writes `error` and `warn` records to stderr, one prefixed line per record, with stack
  traces indented under their first line.
- Replay of `error` records buffered before the plugin loaded.
- Bridging of `api-session/error`, so session-activation failures are recorded even though they never reach
  the logger.
- A rate limiter at 100 messages per 60-second window, with an explicit line reporting how many were dropped.
- One ready line per launch on stdout, so a log without it can be read as predating the bridge.
- Every attachment step wrapped, so the bridge cannot break the boot it exists to explain.

### Fixed

- The published `files` list now includes `cordis.patch.yml`. Without it, an npm install shipped `index.js`
  alone and the bundle row that mounts the plugin was missing.