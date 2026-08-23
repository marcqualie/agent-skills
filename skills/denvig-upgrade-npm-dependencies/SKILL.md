---
name: denvig-upgrade-npm-dependencies
description: Use the denvig cli to upgrade npm dependencies in the current project, always checking the result against dependabot alerts and known CVEs.
license: MIT
compatibility: Requires the `denvig` and `gh` CLI tools to be installed and configured in the project. Compatible with projects that use npm for dependency management.
disable-model-invocation: true
argument-hint: "[patch|minor|all|{{package}}]"
allowed-tools: Read Edit Write AskUserQuestion WebFetch WebSearch Bash(cat package.json) Bash(denvig outdated) Bash(denvig outdated --semver *) Bash(denvig deps why *) Bash(npm view *) Bash(pnpm install) Bash(pnpm upgrade *) Bash(pnpm -r upgrade *) Bash(pnpm list *) Bash(pnpm run *) Bash(pnpm test *) Bash(pnpm exec tsc *) Bash(gh api *) Bash(gh repo view *) Bash(git add *) Bash(git checkout *) Bash(git stash) Bash(git diff *) Bash(git status *) Bash(git commit *) Bash(git push *) Bash(gh pr create *) Bash(gh pr view *)
---
You are an expert software engineer specialized in managing and upgrading npm dependencies in TypeScript projects.
Denvig is a specialised CLI tool that can assist with identifying outdated dependencies.

The user has asked you to upgrade: $ARGUMENTS

Every run follows the same process regardless of what is being upgraded: work out what to upgrade, read the changelogs, apply the upgrade, **check the result against dependabot alerts**, verify the project still works, then commit and open a PR. The vulnerability check is never optional and is never skipped because of the type of upgrade being performed.

## 1. Determine what to upgrade, and record the vulnerability baseline

Before changing anything, record the vulnerability baseline. You need the *pre-upgrade* state to work out what your change actually fixes, so this has to happen first even though the analysis comes later in step 4.

1. Fetch the dependabot alerts using the command in step 4 and keep the full JSON.
2. Run `denvig deps why {{alerted_package}}` for **every** package named in those alerts, and record the versions installed right now.

Without that second pass you cannot honestly say an alert was "resolved by this change" — an alert may already be stale against the current lockfile, and claiming credit for it would be wrong. Note that `denvig deps why` exits non-zero with "Dependency ... not found in this project" when a package is absent; that is a valid answer, not an error, so do not let it abort a loop.

- Assume all dependencies are semver compatible.
- Determine the type of upgrade (patch, patch and minor, major single, named package). Assume patch unless prompted otherwise.
- If upgrading minor, then patch upgrades should also be included.
- If upgrading major, then only a single dependency should be upgraded at a time. Ensure the prompt includes the name of a dependency to upgrade.
- List all the dependencies in package.json.
- Patch and minor dependencies can be identified quickly via `denvig outdated --semver patch --ecosystem npm` and `denvig outdated --semver minor --ecosystem npm`.
- Major versions can be identified with `denvig outdated` command.

When the user names a specific package, the rules in this block take precedence over the semver defaults above:

- Run `denvig deps why {{package}}` first to establish whether it is a direct or transitive dependency, and to record every version currently installed. The version in parentheses in that output is the latest published version, not a second installed copy.
- Upgrade it to the latest available version, crossing majors if necessary, unless something puts the latest version out of reach. If you stop short of the latest version, say why under `## Notes`.
- A blocker is not always in a changelog. For a transitive package the usual ceiling is the semver range its parent declares, which you can read with `npm view {{parent}} dependencies`. Check that before assuming the latest version is reachable, and name the parent and its range when it caps you.
- Both direct and transitive packages run the full process below. Only the mechanism for moving the version differs.

## 2. Research the changelog

- For each dependency that will be updated, you should find the releases/changelog for that dependency.
- You can identify the git repo for a package by running `npm view {{package}} repository.url`.
- Do not run `npm view {{package}} versions` or similar commands that list all versions since you already have that information from the `outdated` command.
- The releases page ({{repository_url}}/releases) or changelog file ({{repository_url}}/blob/main/CHANGELOG.md or similar) should contain the information you need.
- Read the changelog and determine if there are any breaking changes or important notes for the upgrade.
- For a major upgrade, also look for a dedicated migration guide and follow it.
- *Never* attempt to clone a dependency repository locally.

## 3. Apply the upgrade

For a **direct dependency** (it appears in `package.json`):

- Once you are sure a dependency can be safely upgraded, edit the version in package.json by hand, preserving the existing range style (an exact pin stays exact, a `^` range stays a `^` range).
- Run the `pnpm install` command to install the updated packages.
- *Never* run `pnpm upgrade`, `pnpm update` or `npm update` to move a direct dependency. Editing package.json is what keeps the changelog review meaningful.

For a **transitive dependency** (it does not appear in `package.json`):

- Run `pnpm -r upgrade {{package}}` to attempt to upgrade the dependency and any subdependencies.
- Check the diff for `pnpm-lock.yaml` to see what was actually able to be updated.
- Run `denvig deps why {{package}}` again afterwards to confirm which versions are now installed.
- If a version is still pinned below the version you need, identify the parent package that pins it from the `deps why` output. Report that parent under `## Notes`, and if upgrading the parent is in scope of the user's request then upgrade it too.

## 4. Check vulnerabilities (always)

This step runs on **every** invocation of this skill — patch, minor, major, direct or transitive. Never skip it and never state that it was skipped without first attempting the API call below and reporting the actual error.

Fetch the baseline as part of step 1, then do the comparison here once the upgrade is installed.

Resolve the repository slug with `gh repo view --json nameWithOwner -q .nameWithOwner`, then fetch the open alerts:

```bash
gh api "repos/{{owner}}/{{repo}}/dependabot/alerts?state=open&per_page=100" --paginate
```

- The query parameters **must** be part of the URL string. Do not pass them with `-f`/`-F`, because that switches the request to `POST` and the endpoint returns `404`. Getting a `404` from a repository you can otherwise read is almost always this mistake, not a missing permission.
- Keep the full JSON for reference. The fields you need per alert are `number`, `dependency.package.name`, `dependency.relationship`, `security_advisory.severity`, `security_advisory.ghsa_id`, `security_advisory.cve_id`, `security_advisory.summary`, `security_vulnerability.vulnerable_version_range` and `security_vulnerability.first_patched_version.identifier`.
- Several alerts commonly exist for the same package. Treat each alert number separately, because they usually have different patched versions.
- Dependabot sometimes files the *same* advisory twice for one package, once against `package.json` and once against `pnpm-lock.yaml`. Count and list alert numbers, so a duplicated advisory counts twice, but say under `## Notes` which ones are duplicate pairs so the headline is not read as more distinct advisories than were really fixed.

To decide whether an alert is resolved, run `denvig deps why {{alerted_package}}` again after the upgrade and compare against the baseline you took in step 1. An alert is **resolved** only when no remaining installed version falls inside `vulnerable_version_range`. If any installed copy is still in range, the alert is **not** resolved. A package that has disappeared from the tree entirely also resolves its alerts.

A package can be installed at more than one version at once. Every copy has to clear the range, not just the one that moved.

- Check every alerted package this way, not just the one you upgraded.
- An upgrade will often resolve alerts for packages other than the one named, because transitive dependencies move or drop out with it. Include all of them.
- List every resolved alert under `## Vulnerabilities`, grouped by package, and state the count.
- If an alerted package was moved but is still vulnerable, or an alert relates to a package this run deliberately did not touch, mention it under `## Notes` rather than claiming it as fixed. Name the parent that pins it, from the `deps why` output.
- Do not claim a count you have not verified against the version data. It is better to report fewer alerts accurately than to assume a whole package's alerts are cleared.
- Dependabot will not close the alerts until the change is merged, so report them as resolved by this change rather than as already closed.
- Copy the GHSA id, CVE id and summary out of the alert JSON when writing them up. They are not the sort of thing to recall from memory, and a plausible-looking GHSA id that belongs to a different advisory is worse than no link at all.
- If the origin remote is not a GitHub repository, or the API call genuinely fails, omit the `## Vulnerabilities` section and state the specific error under `## Notes`.

## 5. Verify

Read the `scripts` block in `package.json` and run every check it defines that is relevant to an upgrade, typically build, lint and test, alongside `pnpm install` itself. If the project is TypeScript and has no typecheck script, run `pnpm exec tsc --noEmit` as well. Do not invent checks the project has no tooling for.

Record the outcome of each check. If any of them fail, fix the failure or report it clearly rather than claiming a clean run.

## 6. Commit and open a PR

Once you have completed the above steps for each package you should summarize all the changes in the format below.

- If upgrading multiple dependencies then {{git_commit_message}} should be: `Update {{count}} (patch|minor) dependencies`
- If upgrading a single dependency then {{git_commit_message}} should be: `Upgrade {{package}} from {{old version}} to {{new version}}`

If the current git state is not clean then stash the current state, alerting the user to this at the end of the process.
If the current branch is `main` then check the git remote `origin` to make the following choice:
- If `origin` is a github.com remote then create a new branch called `denvig/upgrade-{{count}}-{{type}}-packages`, or `denvig/upgrade-{{package}}` when the user named a single package.
- If `origin` is any other provider then use your AskUserQuestion tool to ask the user if they want to create a new branch or continue on main.
- You should always ask for clarification when github is not detected, even if auto mode is active.

If the current branch is not `main`, stay on it and commit there rather than creating another branch.

Create a git commit with the below summary format if there is at least one dependency upgraded. Examples are provided below for patch, minor and major upgrades.

If the branch is not `main`, then a GitHub PR should be created using `gh pr create --draft --assignee @me` to create a draft pull request with with the title as the git commit message and the body as the summary of changes.
Open the Pull Request in the browser using `gh pr view {id} -w`.

```markdown
{{git_commit_message}}

## Vulnerabilities

Resolves {{count}} dependabot alerts.

- {{package}} [{{old_version}} -> {{new_version}}]
  - [#{{alert_id}}]({{full_link_to_alert}}) [{{cve_id}}]({{full_link_to_advisory}}) ({{severity}}) {{alert_summary}}

## Package Updates

- {{package}}: [{{old_version}} -> {{new_version}}]({{link_to_changelog_or_diff}})
  - {{summary_of_changes_from_changelog}}

## Code Changes

{{details_of_code_modifications_made}}

## Checks

{{list_of_checks_performed_after_upgrade}}
```

Notes on the template:

- The alert link is `https://github.com/{{owner}}/{{repo}}/security/dependabot/{{alert_id}}` and the advisory link is `https://github.com/advisories/{{ghsa_id}}`. Use the GHSA id as the link text when an alert has no CVE id.
- When a package leaves the dependency tree altogether, write the version pair as `[{{old_version}} -> removed]`.
- When a package had several copies installed and only some moved, write the pair for the copy that changed and explain the rest under `## Notes`.
- `## Package Updates` covers the packages you deliberately upgraded, with their changelogs. Transitive packages that merely moved as a side effect belong under `## Vulnerabilities`, not here, unless they required a code change.
- Use the single-dependency commit message form when the user named one package, even if upgrading it moved many transitive packages.
- Omit the `## Vulnerabilities` section entirely when the upgrade resolved no alerts, and add a check line confirming that the alerts were read and none were affected.
- The vulnerability check line always takes the form `{{resolved}} of the {{open}} open alerts are resolved by this upgrade`, however large the remainder is. Alerts left open for reasons unrelated to this upgrade are normal and belong under `## Notes`, ideally saying which single upgrade would clear the most of them.

## Example Commit Message

### Patch

```markdown
Update 2 patch dependencies

## Package Updates

- react: [19.1.0 -> 19.1.1](https://github.com/facebook/react/releases/tag/v19.1.1)
  - Fixed Owner Stacks to work with ES2015 function.name semantics
- zod: [4.1.4 -> 4.1.7](https://github.com/colinhacks/zod/compare/v4.1.4...v4.1.7)
  - Update z.function() type to support array input (#5170)
  - Updated docs

## Code Changes

All patch versions with no breaking changes or code modifications required.

## Checks

- ✅ Checked changelogs for breaking changes
- ✅ Checked dependabot alerts, none of the 3 open alerts are affected by these upgrades
- ✅ Lint passes after upgrade
- ✅ All tests pass after upgrade
- ✅ No code modifications were necessary
```


### Minor

```markdown
Update 2 minor dependencies

## Package Updates

- typescript: [5.8.2 -> 5.9.2](https://github.com/microsoft/TypeScript/compare/v5.8.2...v5.9.2)
  - Release announcement: https://devblogs.microsoft.com/typescript/announcing-typescript-5-9/
  - Support for `--module node20`
  - Support for `import defer`
- biome: [2.1.0 -> 2.2.0](https://github.com/biomejs/biome/compare/%40biomejs/biome%402.1.0...%40biomejs/biome%402.2.0)
  - The noRestrictedImports rule has been enhanced with a new patterns option
  - Improved useExhaustiveDependencies to better handle complex hook calls

## Code Changes

- Ran `biome migrate` to apply necessary code modifications for biome 2.2.0
- Removed unused deps from useEffect hooks that don't actually require them

## Notes

- Some `uesEffect` hooks have ignore rules which can be removed with the new biome version.

## Checks

- ✅ Checked changelogs for breaking changes
- ✅ Checked dependabot alerts, none of the 3 open alerts are affected by these upgrades
- ✅ Lint passes after upgrade
- ✅ All tests pass after upgrade
- ✅ Code modifications applied where necessary
```

### Major with vulnerabilities

```markdown
Upgrade yaml from 1.10.2 to 2.8.3

## Vulnerabilities

Resolves 2 dependabot alerts.

- yaml [1.10.2 -> 2.8.3]
  - [#10](https://github.com/marcqualie/denvig/security/dependabot/10) [CVE-2026-33532](https://github.com/advisories/GHSA-48c2-rrv3-qjmp) (high) yaml is vulnerable to Stack Overflow via deeply nested YAML collections
  - [#11](https://github.com/marcqualie/denvig/security/dependabot/11) [GHSA-f9xv-q969-pqx4](https://github.com/advisories/GHSA-f9xv-q969-pqx4) (medium) yaml is vulnerable to prototype pollution when parsing untrusted documents

## Package Updates

- yaml: [1.10.2 -> 2.8.3](https://github.com/eemeli/yaml/compare/v1.10.2...v2.8.3)
  - Rewritten public API, `safeLoad` replaced by `parse`
  - Add trailingComma ToString option for multiline flow formatting
  - Catch stack overflow during node composition

## Code Changes

- Replaced `YAML.safeLoad(...)` with `YAML.parse(...)` in `src/config.ts`

## Notes

- `@some/tool` still pins `yaml@1.10.2` in its own dependency tree, so alert [#12](https://github.com/marcqualie/denvig/security/dependabot/12) remains open until that package is upgraded.

## Checks

- ✅ Checked changelogs for breaking changes
- ✅ Checked dependabot alerts, 2 of the 3 open alerts are resolved by this upgrade
- ✅ Lint passes after upgrade
- ✅ All tests pass after upgrade
- ✅ Code modifications applied where necessary
```
