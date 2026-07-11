# Local Fork Guide

This is the gyoz-ai fork of herdr. Fork-only features live on `master`, ahead of upstream.

## Layout

- `origin` — git@github.com:gyoz-ai/herdr.git (this fork; the only push target)
- `upstream` — git@github.com:ogulcancelik/herdr.git (never pushed to)
- `master` is a one-feature-one-commit ledger rebased onto upstream's default branch
  via the fork-sync workflow. Each fork feature is exactly one commit.

## Commit conventions

- Fixes to an existing fork feature fold into its ledger commit:
  `git commit --no-gpg-sign --fixup=<sha>` then autosquash on the next rebase.
- All commits use `--no-gpg-sign`.
- Push to `origin` only, always with `--force-with-lease` (the ledger rewrites history on rebase).

## Build

```sh
PATH="$(brew --prefix zig@0.15)/bin:$PATH" cargo build --release --locked
```

## Test

```sh
cargo nextest run
```

If a gpg agent prompt hangs the worktree tests, disable signing for the run:

```sh
GIT_CONFIG_COUNT=2 GIT_CONFIG_KEY_0=commit.gpgsign=false GIT_CONFIG_KEY_1=tag.gpgsign=false cargo nextest run
```

## Local install

The active binary is a plain-file copy at `/opt/homebrew/bin/herdr` that shadows the
brew 0.7.3 formula. There is no symlink and no `cargo install`.

Activate newly built code:

```sh
cp target/release/herdr /opt/homebrew/bin/herdr
```

then restart herdr.

Do NOT run `brew link --overwrite herdr` — that restores the upstream bottle instead
of the fork build.

## Syncing with upstream

Use the fork-sync skill workflow: it fetches upstream, rebases the feature ledger onto
the upstream default branch, reruns the build+test gate, and pushes to origin with
`--force-with-lease` only after explicit confirmation.
