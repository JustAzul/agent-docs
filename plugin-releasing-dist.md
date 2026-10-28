# Releasing a plugin published from a `dist` branch

For a repository whose plugin is installed from a `dist` branch that CI builds,
and whose version lives in a `VERSION` file. Read the values this guide leaves
as placeholders from the repository: the version from `VERSION`, the marketplace
and plugin names from `.claude-plugin/marketplace.json`, the repository from
`git remote get-url origin` (`<owner>/<repo>`).

A push to `main` publishes nothing. Only a release commit does.

## When

Cut a release only when the owner asks. Unreleased work stays on `main` and
reaches no install, however many commits pile up. Do not release on your own
judgment because a fix landed, and do not change `VERSION` outside a release.

## Choosing the version

`VERSION` holds one line: a [Semantic Versioning 2.0.0](https://semver.org)
version with no pre-release part. The public contract is whatever the
repository documents: commands, flags and options, exit codes, JSON and other
output shapes, persisted formats, hooks, tools and interfaces.

- Patch (`1.4.2` to `1.4.3`): a compatible fix.
- Minor (`1.4.2` to `1.5.0`): new behaviour that keeps the contract, or a new
  command, hook, tool or option.
- Major (`1.4.2` to `2.0.0`): a change that breaks stored data, an option, or
  an output shape. While the major is `0`, such a change bumps the minor.
- `1.0.0` declares the contract stable. Release it only when the owner asks for
  it, never as the next bump after an incompatible change.
- No pre-releases on the dist channel. Every release moves `dist`, so every
  install would take a pre-release as the current version. The release gate
  refuses a version with a pre-release part and nothing is published.

## How

1. Start from an up-to-date `main` whose CI is green, and run the repository's
   local checks.
2. Make one commit that changes only `VERSION`, to the new version and nothing
   else, with the subject exactly `chore: release <version>`. No `v` prefix, no
   suffix, no body needed.
3. Push to `main` in the normal way, as a plain push or a fast-forward, so the
   release commit reaches `main` unchanged. A squash or a merge commit changes
   the subject and the diff, and nothing publishes. The release commit must be
   the last commit of the push: CI looks only at the commit the push ends on, so a release
   commit followed by another commit in the same push publishes nothing.
   This push publishes the release: make it only after the owner approves it.
4. CI publishes. Its gate accepts a commit only when the subject is exactly
   that, the diff against its first parent touches only `VERSION`, and
   `VERSION` equals the version in the subject. A separate publish job then, in
   this order: builds the plugin tree, tags the build commit `v<version>`,
   creates the GitHub release with notes from the commits since the previous
   release, and moves `dist` to the tagged commit last. Installs follow the
   `dist` branch; the GitHub release tags stay `v<version>`.
5. Release one version at a time. Wait for the publish job to finish before
   pushing the next release. Publish runs queue and none is cancelled once
   started, but GitHub keeps only one waiting run per group, so a third release
   queued behind two others replaces the second. A dropped run is rerun, below.

## After the release

An installed copy updates only when the version in the `dist` manifest
changes, and each version is cached in a folder of its own.

1. `claude plugin marketplace update <marketplace>`
2. `claude plugin update <plugin>@<marketplace>`
3. `/reload-plugins` in each running session.
4. Check `~/.claude/plugins/cache/<marketplace>/<plugin>/<version>/` exists.

With auto-update on for the marketplace, Claude Code refreshes it in the
background after a session starts and installs the new version, so steps 1 and
2 by hand are only for checking a release at once.

To check the publication: `git ls-remote origin refs/heads/dist
refs/tags/v<version>` shows the same commit for both, and `gh release view
v<version> --repo <owner>/<repo>` shows the release.

## Failure and recovery

- A failed or dropped release run: rerun the run of the release commit's push
  (`gh run rerun <run-id>`). The rerun runs at the same commit and finishes
  whatever is missing, which can be the tag, the release, or the move of
  `dist`. An existing tag is reused after the script checks that it points at a
  dist build of that commit; it is never rebuilt or moved.
- A rerun of an older release after a newer one is already on `dist` creates
  its tag and release and leaves `dist` alone. `dist` never goes back past a
  newer release.
- The run stops, changing nothing, when the release lookup fails with anything
  but "not found", when origin's `dist` branch cannot be listed, when a tag
  `v<version>` exists on anything other than a dist
  build of this commit, or when `dist` holds a build the checkout cannot place
  in its history. Read the message, fix the cause, rerun. Do not delete or move
  a tag to get past it.
- A pushed tag is never moved or deleted, and a published release is never
  edited into a different build.
- The gate printed `release=false` for a commit meant as a release: nothing was
  tagged. Read the reason in the check job's log. A corrected release commit
  may reuse the version, because nothing was published under it.
- Rolling back: a bad release is not undone. Restore the previous code in a new
  commit and release it under a higher version. An install updates whenever the
  version differs from the one it has, so a lower number would also install, but
  nothing in the gate or in the publish step compares versions. Keep every
  release higher than the one in `VERSION` before it; that is on you.
