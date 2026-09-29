---
name: denvig-upgrade-github-actions
description: Use the denvig cli to upgrade GitHub Actions in the current project, pinning every action to a full commit SHA with a version comment.
license: MIT
compatibility: Requires the `denvig` and `gh` CLI tools to be installed and configured in the project. Compatible with projects that define GitHub Actions workflows in `.github/workflows`.
disable-model-invocation: true
argument-hint: "[all|pin|{{action}}]"
allowed-tools: Read Edit Write Grep Glob AskUserQuestion WebFetch WebSearch Bash(denvig outdated) Bash(denvig outdated *) Bash(denvig deps list *) Bash(denvig deps why *) Bash(gh api *) Bash(gh release view *) Bash(gh repo view *) Bash(git remote *) Bash(git fetch *) Bash(git branch *) Bash(git switch *) Bash(git log *) Bash(git add *) Bash(git checkout *) Bash(git stash) Bash(git diff *) Bash(git status *) Bash(git commit *) Bash(git push *) Bash(gh pr create *) Bash(gh pr view *)
---
You are an expert software engineer specialized in maintaining GitHub Actions workflows and CI supply chain security.
Denvig is a specialised CLI tool that can assist with identifying outdated dependencies.

The user has asked you to upgrade: $ARGUMENTS

Every run follows the same process regardless of what is being upgraded: work out what to upgrade, read the changelogs, apply the upgrade **pinned to a commit SHA**, verify the workflows, then commit and open a PR.

## Pin every action to a commit SHA

This is the most important rule in this skill. Tags on a GitHub repository are mutable: anyone with write access to an action's repository, or anyone who compromises it, can move `v4` or even `v4.2.0` to point at different code, and every workflow referencing that tag runs it on the next push with your repository's secrets. A full commit SHA cannot be moved.

Every `uses:` reference this skill writes must be a full 40 character commit SHA followed by a comment containing the exact version tag it corresponds to:

```yaml
- uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
```

- *Never* write a tag (`@v7`, `@v7.0.1`), a branch (`@main`) or a short SHA as the ref.
- The comment is the exact tag name as the action's repository spells it, including or omitting the `v` prefix to match. It is always a full version, never a floating major such as `# v7`.
- Keep a single space either side of the `#`. Dependabot and Renovate both recognise this format and will keep the comment in step with the SHA if either is enabled later.
- Pinning applies to reusable workflows too: `uses: owner/repo/.github/workflows/build.yml@{{sha}} # v2.1.0`.
- Local references (`uses: ./.github/actions/setup`) and container references (`uses: docker://...`) have no git ref to pin and are left alone.

## 1. Determine what to upgrade

Find every file that can reference an action. That is every `*.yml` and `*.yaml` under `.github/workflows/`, and every `action.yml` or `action.yaml` for composite actions, usually under `.github/actions/`. Grep them for `uses:` and record every reference along with its file and current ref.

Run `denvig outdated --ecosystem actions` to list the actions with a newer release. Add `--json` when you need the per-file detail: each entry has a `versions` array giving the `resolved` version, the `specifier` as written and the `source` file it came from.

- `denvig outdated` resolves a SHA-pinned reference from the SHA itself, not the comment, so for an outdated action its `Current` version is the truth and an existing `# vX.Y.Z` comment that disagrees is stale.
- A floating tag such as `@v4` is reported as the release it currently points at, such as `4.4.0`.
- Actions that are already up to date never appear in `denvig outdated`, and `denvig deps list` shows only the raw ref, never a version. For those, work out the version with the GitHub API as described in step 3.
- Existing SHA pins on up-to-date actions can also carry stale comments. The check in step 4 finds them. Fix any stale comments as part of this run and list them under `## Notes`.
- The `Wanted` column is always blank for actions, and `--semver patch` or `--semver minor` report nothing even when patch or minor releases exist. Do not rely on them. Compare `Current` against `Latest` yourself.
- The same action is often referenced from several files, sometimes at different versions. Treat every reference as in scope and move them all to the same version.

Decide the scope from the arguments:

- **No argument or `all`**: upgrade every outdated action to its latest release, crossing majors. Action majors are routinely just a Node.js runtime bump, so they are not held back the way npm majors are, but each still needs its changelog read in step 2. Also pin every remaining tag or branch reference in the repository at its current version, even for actions that are already up to date.
- **`pin`**: upgrade nothing. Pin every tag or branch reference at the version it resolves to today, and fix any stale comments. Step 2 is skipped because no version moves.
- **A named action**, such as `actions/checkout`: upgrade and pin every reference to that action only. Leave other actions alone, even unpinned ones, and mention how many unpinned references remain under `## Notes`.

If you stop short of the latest version for any action, say why under `## Notes`.

## 2. Research the changelog

- For each action that will be upgraded, read the releases between the current and the new version. `gh api "repos/{{owner}}/{{repo}}/releases?per_page=100"` lists them, and `gh release view {{tag}} -R {{owner}}/{{repo}}` shows the notes for one.
- Some actions publish no GitHub releases. Fall back to the `CHANGELOG.md` in the repository, then to `https://github.com/{{owner}}/{{repo}}/compare/{{old_tag}}...{{new_tag}}` and summarise the commits.
- For an action referenced by a subpath, such as `github/codeql-action/init`, the repository is the first two segments, `github/codeql-action`.
- Look specifically for the things that break workflows:
  - **Renamed, removed or newly required inputs.** Compare the `with:` keys each workflow passes against the `inputs:` in the new version's `action.yml`. Read it with `gh api "repos/{{owner}}/{{repo}}/contents/action.yml?ref={{new_tag}}" -H "Accept: application/vnd.github.raw"` (some actions use `action.yaml`, and subpath actions keep it under their subpath). Use this rather than WebFetch, which summarises the file and can drop inputs, and rather than `curl`.
  - **Changed or removed outputs** referenced elsewhere via `steps.{{id}}.outputs.*`.
  - **Runtime bumps**, such as `node20` to `node24`. These need a minimum runner version, which matters for self-hosted runners and GitHub Enterprise Server, and are usually the only change in an action major.
  - **Changed defaults** for inputs the workflow does not set, such as `actions/checkout` changing its credential persistence or `actions/upload-artifact` changing overwrite and retention behaviour.
- For a major upgrade, also look for a dedicated migration guide in the README or release notes and follow it.
- *Never* attempt to clone an action repository locally.

## 3. Apply the upgrade

Resolve the commit SHA for each target tag:

```bash
gh api "repos/{{owner}}/{{repo}}/commits/{{tag}}" --jq .sha
```

- Always use the `commits/{{tag}}` endpoint. Do **not** use `git/ref/tags/{{tag}}`: for an annotated tag it returns the SHA of the tag *object* rather than the commit, and pinning to that produces a reference that fails to resolve when the workflow runs.
- The endpoint returns `422` when the tag does not exist, which usually means the repository spells its tags differently, such as `1.2.0` rather than `v1.2.0`. Look at the releases list for the real tag name rather than guessing.
- Confirm the result is exactly 40 hexadecimal characters before writing it.
- When pinning an action at its current version, pin to the full version tag (`v4.4.0`), not the floating major tag (`v4`), even though today they point at the same commit. For an outdated action, denvig's `Current` column gives you that version. For an up-to-date one, resolve the floating tag's SHA with the same endpoint, then find the release whose tag resolves to that same SHA. Start with `gh api "repos/{{owner}}/{{repo}}/releases/latest" --jq .tag_name`, which is usually it. If no full version tag shares the SHA, pin the SHA anyway, keep the floating tag in the comment, and say so under `## Notes`.

Edit every in-scope `uses:` line to `{{owner}}/{{repo}}@{{sha}} # {{tag}}`. Preserve the surrounding indentation, list style (`- uses:` or `uses:`) and any other comment content on the line.

Apply any workflow changes the changelog calls for, such as renaming an input, in the same edit and record them under `## Code Changes`.

## 4. Verify

There is no lockfile and no install step, so verification is about proving the references are correct:

- Run `denvig outdated --ecosystem actions` again. Every action upgraded in this run must be absent from the output.
- Grep every workflow and composite action file for `uses:` again, and confirm every in-scope reference is a 40 character SHA followed by a `# {{tag}}` comment. Report any remaining tag references that were out of scope under `## Notes`.
- For every distinct pinned `{{owner}}/{{repo}}@{{sha}} # {{tag}}` in the repository, not just the ones you changed, run `gh api "repos/{{owner}}/{{repo}}/commits/{{tag}}" --jq .sha` and confirm the result equals the pinned SHA. This is the only check that proves each comment is true, and it also catches stale comments on pins that already existed. Run each lookup as its own `gh api` command rather than wrapping them in a shell loop, so they match the pre-approved tools and don't prompt. `denvig` cannot do it, because once an action is up to date it no longer reports a version for it.
- If the project defines checks that lint the workflow files, run those as well. Do not invent checks the project has no tooling for.

The real verification is the workflows running on the pull request. Point this out in the final message to the user. Any job that is gated to `main`, tags or a schedule, whether by the workflow's `on:` triggers or by a job-level `if:`, will not be exercised by the PR. Name those jobs under `## Notes`, and say whether each action they use is also used by a job that does run on the PR.

Record the outcome of each check. If any of them fail, fix the failure or report it clearly rather than claiming a clean run.

## 5. Commit and open a PR

Once you have completed the above steps for each action you should summarize all the changes in the format below.

- If upgrading multiple actions then {{git_commit_message}} should be: `Update {{count}} GitHub Actions`
- If upgrading a single action then {{git_commit_message}} should be: `Upgrade {{action}} from {{old version}} to {{new version}}`
- If only pinning, with no version changes, then {{git_commit_message}} should be: `Pin {{count}} GitHub Actions to commit SHAs`
- `{{count}}` is the number of distinct actions, not references. When updating, count only the actions whose version moved. Actions that were only pinned are listed under `## Pinned` but are not counted in the title or branch name.

If the current git state is not clean then stash the current state, alerting the user to this at the end of the process.
If the current branch is `main` then check the git remote `origin` to make the following choice:
- If `origin` is a github.com remote then create a new branch called `denvig/upgrade-{{count}}-actions`, `denvig/pin-{{count}}-actions` when only pinning, or `denvig/upgrade-{{owner}}-{{repo}}` when the user named a single action.
- If `origin` is any other provider then use your AskUserQuestion tool to ask the user if they want to create a new branch or continue on main.
- You should always ask for clarification when github is not detected, even if auto mode is active.

If the current branch is not `main`, stay on it and commit there rather than creating another branch.

Create a git commit with the below summary format if at least one reference was changed. Examples are provided below.

If the branch is not `main`, then a GitHub PR should be created using `gh pr create --draft --assignee @me --base main` to create a draft pull request with the title as the git commit message and the body as the summary of changes, without its first line. Pass `--base` explicitly, because a branch created from `origin/main` tracks it and `gh` will otherwise refuse to guess.
Open the Pull Request in the browser using `gh pr view {id} -w`.

```markdown
{{git_commit_message}}

## Action Updates

- {{action}}: [{{old_version}} -> {{new_version}}]({{link_to_release_or_compare}})
  - {{summary_of_changes_from_changelog}}

## Pinned

- {{action}} {{version}}

## Code Changes

{{details_of_code_modifications_made}}

## Notes

{{anything_the_reviewer_should_know}}

## Checks

{{list_of_checks_performed_after_upgrade}}
```

Notes on the template:

- `## Action Updates` covers the actions whose version moved. Omit it for a pin-only run.
- `## Pinned` covers the actions whose version did not move but whose reference was converted from a tag or branch to a SHA. Omit it when there were none.
- Omit `## Notes` when there is nothing to say.
- Write versions without the `v` prefix in the version pairs, matching the other denvig skills, but keep the exact tag in the `uses:` comment.
- Every run includes a check line stating how many references are now SHA pinned, in the form `{{pinned}} of {{total}} action references are pinned to commit SHAs`.

## Example Commit Message

### All

```markdown
Update 3 GitHub Actions

## Action Updates

- actions/checkout: [4.4.0 -> 7.0.1](https://github.com/actions/checkout/compare/v4.4.0...v7.0.1)
  - Runs on the `node24` runtime, requiring runner v2.327.1 or later
  - Credentials are persisted to a separate file rather than `.git/config`
- actions/setup-node: [3.8.1 -> 7.0.0](https://github.com/actions/setup-node/compare/v3.8.1...v7.0.0)
  - Runs on the `node24` runtime
  - Automatic caching is enabled when `packageManager` is set in `package.json`
- pnpm/action-setup: [4.4.0 -> 6.1.0](https://github.com/pnpm/action-setup/compare/v4.4.0...v6.1.0)
  - The `version` input is optional and read from `packageManager` when omitted

## Pinned

- denoland/setup-deno 2.0.3

## Code Changes

- Removed the `version` input from `pnpm/action-setup` in `.github/workflows/ci.yml`, since it duplicated `packageManager`

## Notes

- `.github/workflows/release.yml` only runs on tag pushes, so its references will not be exercised by this PR.

## Checks

- ✅ Checked changelogs for breaking changes
- ✅ Checked the `with:` inputs of every upgraded action against its new `action.yml`
- ✅ `denvig outdated` reports no remaining upgrades
- ✅ Every pinned SHA matches the commit its comment's tag resolves to
- ✅ 9 of 9 action references are pinned to commit SHAs
```

### Pin only

```markdown
Pin 2 GitHub Actions to commit SHAs

## Pinned

- actions/checkout 7.0.1
- actions/setup-node 7.0.0

## Code Changes

No workflow changes were necessary.

## Notes

- `actions/checkout` in `.github/workflows/deploy.yml` was already pinned, but its comment read `# v6.1.0` for a SHA that resolves to 7.0.1. The comment has been corrected.

## Checks

- ✅ Every pinned SHA matches the commit its comment's tag resolves to
- ✅ 5 of 5 action references are pinned to commit SHAs
```

### Single action

```markdown
Upgrade actions/upload-artifact from 3.2.2 to 7.0.1

## Action Updates

- actions/upload-artifact: [3.2.2 -> 7.0.1](https://github.com/actions/upload-artifact/compare/v3.2.2...v7.0.1)
  - Artifacts are immutable, and uploading to an existing name fails unless `overwrite: true` is set
  - Artifacts uploaded by v4 and later cannot be downloaded by `actions/download-artifact` v3
  - Runs on the `node24` runtime

## Code Changes

- Gave the per-platform uploads in `.github/workflows/build.yml` distinct names, since they previously appended to one shared artifact
- Upgraded the matching `actions/download-artifact` reference to 7.0.0 so it can read the new artifacts

## Notes

- 4 other action references in this repository are still referenced by tag. Run this skill with `pin` to convert them.

## Checks

- ✅ Checked changelogs for breaking changes
- ✅ Checked the `with:` inputs of every upgraded action against its new `action.yml`
- ✅ `denvig outdated` reports no remaining upgrades for actions/upload-artifact
- ✅ Every pinned SHA matches the commit its comment's tag resolves to
- ✅ 2 of 6 action references are pinned to commit SHAs
```
