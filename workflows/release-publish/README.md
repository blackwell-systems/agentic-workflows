# Release-Publish

Use [github-release-engineer](https://github.com/blackwell-systems/github-release-engineer) to tag, release, and verify a GitHub release, then [homebrew-formula-updater](https://github.com/blackwell-systems/homebrew-formula-updater) to publish the new version to your Homebrew tap.

## Why

A GitHub release and a Homebrew formula update are two separate operations on two separate repos, but they share all the same inputs: version, download URLs, and checksums. Doing them sequentially by hand means copy-pasting SHA256 values and URLs under time pressure after a CI run. The release engineer resolves those values automatically and hands them off directly — the formula updater never needs to download anything or ask for input it already has.

## The loop

```
/release
      │
      ▼
pre-flight (auth, clean tree, branch, ahead/behind)
      │
      ▼
changelog promoted, tag created, pushed
      │
      ▼
CI watched to completion
      │
      ▼
release artifacts verified (checksums.txt + platform binaries)
      │
      ▼
/homebrew-formula-updater (version, URLs, checksums passed from release engineer)
      │
      ▼
formula updated, diff confirmed, committed, pushed to tap
      │
      ▼
brew update && brew upgrade <tool>
```

## Prerequisites

- `gh` CLI authenticated: `gh auth status`
- Goreleaser (or equivalent) configured in CI to attach a `checksums.txt` asset to the release — this is how the formula updater resolves SHA256 values without downloading binaries locally
- Tap repo accessible: cloned locally or reachable for cloning via `gh`

## How to run it

1. **Install both skills** — copy `github-release-engineer.md` and `homebrew-formula-updater.md` into `~/.claude/commands/`.

2. **Configure the release engineer** — in the project repo, ensure `CHANGELOG.md` has an `[Unreleased]` section with entries for the pending release.

3. **Configure the formula updater** — ensure your Homebrew tap is cloned locally. The updater checks `TAP_REPO` env → `~/code/<tap-name>` → `/opt/homebrew/Library/Taps/<org>/<tap-name>` → clones if none found.

4. **Run the release** — from the project repo:
   ```
   /release
   ```
   The release engineer detects the version from the changelog, confirms with you, then runs the full pipeline. At the end it invokes `/homebrew-formula-updater` automatically.

5. **Confirm the formula diff** — the formula updater shows `git diff` before committing. Verify the version, URLs, and SHA256 values look correct, then confirm.

6. **Verify locally** — after the tap push:
   ```bash
   brew update && brew upgrade <tool>
   <tool> --version
   ```

## What each skill handles

| Concern | Handled by |
|---|---|
| Pre-flight (auth, tree, branch) | release engineer |
| Changelog promotion | release engineer |
| Tag creation and push | release engineer |
| CI monitoring and failure diagnosis | release engineer |
| Artifact wait and release verification | release engineer |
| Checksum resolution | formula updater (from release asset) |
| Formula version / URL / SHA256 update | formula updater |
| Tap commit and push | formula updater |

## When CI fails mid-release

The release engineer diagnoses failures from `gh run view --log-failed`, proposes a fix, and waits for confirmation before applying it. If the fix requires a new commit, it retags safely — checking whether a non-draft release already exists before deleting and recreating the tag.

The formula updater is only invoked after CI passes and the release is verified. If CI fails, the formula updater never runs.

## Checksum sourcing fallback chain

The formula updater resolves checksums in this order:

1. `checksums.txt` (or equivalent) attached to the GitHub release — primary, requires no manual input
2. Checksums passed directly from the release engineer's completion report — fallback if no checksum file is attached
3. Ask the user — last resort

With goreleaser's default configuration, option 1 is always available.
