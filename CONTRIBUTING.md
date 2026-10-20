# Contributing

## What this plugin is

Two source files and a bundle patch. The review bar is narrow: **stay out of the boot path, and stay honest
about what you can and cannot see.**

## Ground rules

1. **It must never break the boot.** Every attachment step is wrapped in `try`. A change that can throw out of
   `apply(ctx)` is a regression, because the whole point is to explain a failing startup.
2. **No module-scope imports.** The module deliberately imports nothing, so the loader has nothing to resolve
   and nothing can fail to install. Keep it that way; `ctx` is the only dependency.
3. **Every emitted line carries the prefix.** A supervisor may parse stderr to decide which plugin broke a
   boot. Unprefixed lines could change that verdict.
4. **Say what is missing.** If a record type is not captured, or a window is lost, document it rather than
   implying completeness.

## Local checks

There is no build step. Before opening a pull request:

```powershell
node -e "import('./index.js').then(m => console.log(Object.keys(m)))"
npm pack --dry-run          # confirm cordis.patch.yml is in the payload
```

CI runs the same manifest check, loads the module, packs it, and fails if the bundle patch is missing from the
tarball.

## Pull requests

- One logical change per pull request.
- Commit subjects follow the conventional form used across these repositories: `fix(scope):`, `feat(scope):`,
  `docs(scope):`, `test(scope):`.
- Describe what you observed, not only what you changed.

## Reporting a bug

Include: the harness version, how the harness was launched, whether the `[harness-log]` ready line appears, and
the lines around the failure. **Read the excerpt before pasting it** — bridged output can contain prompt text
and file paths.

## License

Contributions are accepted under the MIT license in [LICENSE](LICENSE).