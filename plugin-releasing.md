# Releasing a Claude Code plugin

For plugin repositories whose marketplace entry pins each plugin to a release tag. A release is the only way a change reaches installed copies: commits that land on `main` between releases reach no install.

## When

- Only when the owner asks for one. An agent never cuts a release on its own, and a push to `main` is not a release.
- Pick the version by semver, against the plugin's `.claude-plugin/plugin.json`:
  - patch: a fix that changes no command, hook, tool, option, output shape or stored data;
  - minor: new behaviour, or a new command, hook, tool or option, that leaves the existing ones working;
  - major: a change that breaks stored data, an option, or the shape of a command, hook or tool output.
- Every release raises the version. Installs update only when it changes, so a release under a version that already exists reaches nobody.
- A repository with several plugins releases each one on its own, with its own version and tag.

## How installs read a release

`.claude-plugin/marketplace.json` pins each plugin to its release tag, `<plugin>--v<version>`. A plugin in a subfolder:

```json
{
  "name": "<plugin>",
  "source": {
    "source": "git-subdir",
    "url": "git@github.com:<owner>/<repo>.git",
    "path": "plugins/<plugin>",
    "ref": "<plugin>--v<version>"
  }
}
```

A plugin at the repository root:

```json
{
  "name": "<plugin>",
  "source": {
    "source": "url",
    "url": "git@github.com:<owner>/<repo>.git",
    "ref": "<plugin>--v<version>"
  }
}
```

Between releases the entry keeps naming the last tag, so every install, fresh ones included, gets the released code. The version lives only in `plugin.json`; the marketplace entry carries no `version` field.

## Steps

Run them from the repository root, on a clean checkout of `main` that is up to date with origin.

0. Release only code that is already on origin's `main` with green CI. `git rev-parse HEAD` equals `git rev-parse origin/main`, and every run listed by `gh run list --commit "$(git rev-parse HEAD)" --json workflowName,conclusion` concluded `success`. In a repository without CI, its local test suite passes on that commit. Push unreleased code first and wait for green; never send it in the same push as the release.
1. Name the plugin and the new version, and read where the plugin lives:

   ```sh
   NAME=<plugin>
   VERSION=<new version>
   MKT=.claude-plugin/marketplace.json
   DIR=$(jq -r --arg n "$NAME" '.plugins[] | select(.name == $n) | .source | if type == "object" then .path // "." else . end' "$MKT")
   MARKETPLACE=$(jq -r .name "$MKT")
   echo "$NAME $VERSION $DIR $MARKETPLACE"
   ```

   `DIR` is `.` for a plugin at the repository root.

2. In `$DIR/.claude-plugin/plugin.json`, set `"version"` to `$VERSION`.
3. In `.claude-plugin/marketplace.json`, set that entry's `source.ref` to `$NAME--v$VERSION`.
4. Commit only those two files, as `chore: release $VERSION`; in a repository with several plugins, as `chore: release $NAME $VERSION`. It is the last commit of the release.
5. Check that the ref names the tag the next steps create; `claude plugin tag` does not compare them:

   ```sh
   test "$(jq -r --arg n "$NAME" '.plugins[] | select(.name == $n) | .source.ref' "$MKT")" \
     = "$NAME--v$(jq -r .version "$DIR/.claude-plugin/plugin.json")" && echo ref-ok
   ```

6. Run `claude plugin tag --dry-run "$DIR"`. It must print `Tag: $NAME--v$VERSION` and a `Marketplace entry:` line.

Steps 7 to 9 publish the release. Run them only after the owner approves the push.

7. Run `claude plugin tag "$DIR"`.
8. Run `git push --atomic origin main "$NAME--v$VERSION"`.
9. Run `git ls-remote origin "refs/tags/$NAME--v$VERSION"`; it prints the tag.

## After a release

- Installed copies update on their own. With auto-update on, Claude Code refreshes the marketplace in the background after a session starts and installs the new version; a running session loads it with `/reload-plugins`.
- To update at once: `claude plugin marketplace update $MARKETPLACE`, then `claude plugin update $NAME@$MARKETPLACE`. A plugin installed at project scope updates in each project that installs it, with `--scope project`.
- A project-scope install or update rewrites the project's tracked `.claude/settings.json` in the CLI's key order. When the content is unchanged, restore the file:

  ```sh
  diff <(git show HEAD:.claude/settings.json | jq -S .) <(jq -S . .claude/settings.json) \
    && git checkout -- .claude/settings.json
  ```

- Check the install: `~/.claude/plugins/cache/$MARKETPLACE/$NAME/$VERSION/` exists (on Windows, under the user profile's `.claude\plugins\cache\`), and the plugin's commands and hooks work in a new session.

## When something fails

- The push failed: nothing reached origin, because the push is atomic. Fix the cause and push again.
- The local tag points at the wrong commit and was never pushed: delete it with `git tag -d "$NAME--v$VERSION"`, fix the commit, and tag again.
- An install reports that the remote branch for the tag was not found: the tag is missing from origin. The install keeps the copy it had; push the tag.
- A pushed tag is never moved or deleted. A bad release is fixed by a new release with a higher version.
- To roll back, restore the previous code on `main` and release it under a higher version.

## First release of a repository

- In the first release commit, turn each entry's `source` from a local path such as `"./plugins/<plugin>"` into the pinned object above.
- Point the repository at this doc. The root `AGENTS.md`, and `CLAUDE.md` when the repository has one, carries this line as plain text, never as an `@` import, so an agent reads the doc only when it needs it:

  > Before cutting a release or changing a plugin's version, fetch and follow https://raw.githubusercontent.com/JustAzul/agent-docs/main/plugin-releasing.md (`curl -fsSL`; it always serves the latest version). If the fetch fails, stop and tell the owner; never release without it.
