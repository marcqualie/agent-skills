---
name: denvig-upgrade-rubygems-dependencies
description: Use the denvig cli to upgrade rubygems dependencies in the current project, always checking the result against dependabot alerts and known CVEs.
license: MIT
compatibility: Requires the `denvig` and `gh` CLI tools to be installed and configured in the project. Compatible with projects that use bundler and rubygems for dependency management.
disable-model-invocation: true
argument-hint: "[patch|minor|all|{{gem}}]"
allowed-tools: Read Edit Write AskUserQuestion WebFetch WebSearch Bash(cat Gemfile) Bash(cat Gemfile.lock) Bash(denvig outdated) Bash(denvig outdated --semver *) Bash(denvig deps why *) Bash(bundle install) Bash(bundle update --conservative *) Bash(bundle info *) Bash(bundle list *) Bash(bundle exec *) Bash(bin/rails *) Bash(bin/rake *) Bash(gem info *) Bash(ruby --version) Bash(gh api *) Bash(gh repo view *) Bash(git add *) Bash(git checkout *) Bash(git stash) Bash(git diff *) Bash(git status *) Bash(git commit *) Bash(git push *) Bash(gh pr create *) Bash(gh pr view *)
---
You are an expert software engineer specialized in managing and upgrading rubygems dependencies in Ruby projects.
Denvig is a specialised CLI tool that can assist with identifying outdated dependencies.

The user has asked you to upgrade: $ARGUMENTS

Every run follows the same process regardless of what is being upgraded: work out what to upgrade, read the changelogs, apply the upgrade, **check the result against dependabot alerts**, verify the project still works, then commit and open a PR. The vulnerability check is never optional and is never skipped because of the type of upgrade being performed.

## 1. Determine what to upgrade, and record the vulnerability baseline

Before changing anything, record the vulnerability baseline. You need the *pre-upgrade* state to work out what your change actually fixes, so this has to happen first even though the analysis comes later in step 4.

1. Fetch the dependabot alerts using the command in step 4 and keep the full JSON.
2. Run `denvig deps why {{alerted_gem}}` for **every** gem named in those alerts, and record the versions installed right now.

Without that second pass you cannot honestly say an alert was "resolved by this change" — an alert may already be stale against the current `Gemfile.lock`, and claiming credit for it would be wrong. Note that `denvig deps why` exits non-zero with "Dependency ... not found in this project" when a gem is absent; that is a valid answer, not an error, so do not let it abort a loop.

- Assume all dependencies are semver compatible.
- Determine the type of upgrade (patch, patch and minor, major single, named gem). Assume patch unless prompted otherwise.
- If upgrading minor, then patch upgrades should also be included.
- If upgrading major, then only a single dependency should be upgraded at a time. Ensure the prompt includes the name of a dependency to upgrade.
- List all the dependencies in the `Gemfile` (and the `*.gemspec` if the project ships a gem and the Gemfile only says `gemspec`).
- Patch and minor dependencies can be identified quickly via `denvig outdated --semver patch --ecosystem rubygems` and `denvig outdated --semver minor --ecosystem rubygems`.
- Major versions can be identified with `denvig outdated` command.
- Ruby security releases often add a fourth segment, such as `8.0.5` to `8.0.5.1`. Those versions do not sort as semver, so `denvig outdated` can leave the `Wanted` column blank for them and the upgrade will not show up in the patch list. When a dependabot alert names a gem, trust the alert's `first_patched_version` over the `outdated` output and upgrade to it regardless.

When the user names a specific gem, the rules in this block take precedence over the semver defaults above:

- Run `denvig deps why {{gem}}` first to establish whether it is a direct or transitive dependency, and to record every version currently installed. The version in parentheses in that output is the latest published version, not a second installed copy.
- Upgrade it to the latest available version, crossing majors if necessary, unless something puts the latest version out of reach. If you stop short of the latest version, say why under `## Notes`.
- A blocker is not always in a changelog. For a transitive gem the usual ceiling is the constraint its parent declares. `bundle info {{gem}}` prints a `Reverse Dependencies` block listing every parent and the constraint it declares, which answers this directly. Check that before assuming the latest version is reachable, and name the parent and its constraint when it caps you.
- Ruby itself can be the ceiling. A gem that requires a newer `required_ruby_version` than the project runs is out of reach, so check `ruby --version` against the gem's requirement before attempting the upgrade, and record it under `## Notes` when it blocks you.
- Both direct and transitive gems run the full process below. Only the mechanism for moving the version differs.

## 2. Research the changelog

- For each dependency that will be updated, you should find the releases/changelog for that dependency.
- `bundle info {{gem}}` prints the `Homepage` and `Changelog` for the installed version and is the quickest way to find them. For a gem that is not installed yet, `https://rubygems.org/api/v1/gems/{{gem}}.json` exposes the same thing as `changelog_uri`, `source_code_uri` and `homepage_uri`.
- A `changelog_uri` usually carries an anchor for its own version, such as `.../CHANGELOG.md#v2.8.8`. Strip the anchor before reading, otherwise you get the notes for the version you already have rather than the ones you are moving to.
- Do not run `gem list {{gem}} --remote --all` or similar commands that list all versions since you already have that information from the `outdated` command.
- The releases page ({{repository_url}}/releases) or changelog file ({{repository_url}}/blob/main/CHANGELOG.md or similar) should contain the information you need.
- A patch cut from a maintenance branch may appear in neither. Gems that keep several release lines alive often tag `v3.2.7` off a `3-2-stable` branch, cut no GitHub release for it, and leave the changelog on `main` describing only unreleased work on the newest line. When that happens, read `{{repository_url}}/compare/v{{old_version}}...v{{new_version}}` and summarise the commits, and link that compare view rather than a changelog anchor that does not mention the version.
- Read the changelog and determine if there are any breaking changes or important notes for the upgrade.
- For a major upgrade, also look for a dedicated migration guide and follow it. Rails engines and other framework gems usually keep one in `UPGRADING.md` or the guides site.
- *Never* attempt to clone a dependency repository locally.

## 3. Apply the upgrade

For a **direct dependency** (it appears in the `Gemfile` or the project's `*.gemspec`):

Which of the two mechanisms below applies depends on whether the declared constraint is precise enough to express the move. Unlike npm, a bundler constraint is frequently looser than the locked version, and in that case the `Gemfile` is not what is holding the gem back — the `Gemfile.lock` is.

- When the constraint names the version you are moving off, such as `gem 'sidekiq', '~> 8.1.6'` going to `8.1.7`, edit it by hand and run `bundle install`. Preserve the existing style: an exact pin stays exact, a `~>` constraint stays `~>`, a `>=` constraint stays `>=`. Bundler then moves that one gem, because the locked version no longer satisfies the `Gemfile`.
- When the constraint is already satisfied by the new version, such as `gem 'rack', '~> 3.2'` going from `3.2.6` to `3.2.7`, there is no edit to make and `bundle install` will do nothing. Run `bundle update --conservative {{gem}}` instead, which moves the lockfile entry without dragging its dependencies along.
- If you edited the `Gemfile` and `bundle install` leaves `Gemfile.lock` unchanged, you were in the second case — follow it with `bundle update --conservative {{gem}}`.
- Check `git diff Gemfile.lock` after either mechanism. It should contain only the gems you intended to move. Anything else means the resolver took liberties and needs explaining under `## Notes`.
- *Never* run bare `bundle update` or `bundle update --all` to move a dependency. Editing the constraint and updating one gem at a time is what keeps the changelog review meaningful.

For a **transitive dependency** (it does not appear in the `Gemfile` or gemspec):

- Run `bundle update --conservative {{gem}}` to attempt to upgrade the gem without moving its parents.
- Check the diff for `Gemfile.lock` to see what was actually able to be updated.
- Run `denvig deps why {{gem}}` again afterwards to confirm which version is now installed.
- If it is still pinned below the version you need, identify the parent gem that pins it from the `deps why` output. Report that parent under `## Notes`, and if upgrading the parent is in scope of the user's request then upgrade it too. Dropping `--conservative` for that single gem is the usual next step, but only do it when moving the parent is genuinely in scope.

Bundler installs one version of a gem per lockfile, so unlike npm there is never a second copy left behind. If `denvig deps why` reports more than one version, the project has multiple lockfiles (for example a `gemfiles/` appraisal matrix) and each one has to be handled in turn.

## 4. Check vulnerabilities (always)

This step runs on **every** invocation of this skill — patch, minor, major, direct or transitive. Never skip it and never state that it was skipped without first attempting the API call below and reporting the actual error.

Fetch the baseline as part of step 1, then do the comparison here once the upgrade is installed.

Resolve the repository slug with `gh repo view --json nameWithOwner -q .nameWithOwner`, then fetch the open alerts:

```bash
gh api "repos/{{owner}}/{{repo}}/dependabot/alerts?state=open&per_page=100" --paginate
```

- The query parameters **must** be part of the URL string. Do not pass them with `-f`/`-F`, because that switches the request to `POST` and the endpoint returns `404`. Getting a `404` from a repository you can otherwise read is almost always this mistake, not a missing permission.
- Keep the full JSON for reference. The fields you need per alert are `number`, `dependency.package.name`, `dependency.package.ecosystem`, `dependency.relationship`, `security_advisory.severity`, `security_advisory.ghsa_id`, `security_advisory.cve_id`, `security_advisory.summary`, `security_vulnerability.vulnerable_version_range` and `security_vulnerability.first_patched_version.identifier`.
- A polyglot repository will return alerts for several ecosystems at once. Filter on `dependency.package.ecosystem == "rubygems"` for this skill, and say under `## Notes` how many alerts belong to other ecosystems so the totals are not read as ruby-only.
- Several alerts commonly exist for the same gem. Treat each alert number separately, because they usually have different patched versions.
- Dependabot sometimes files the *same* advisory twice for one gem, once against `Gemfile` and once against `Gemfile.lock`. Count and list alert numbers, so a duplicated advisory counts twice, but say under `## Notes` which ones are duplicate pairs so the headline is not read as more distinct advisories than were really fixed.

To decide whether an alert is resolved, run `denvig deps why {{alerted_gem}}` again after the upgrade and compare against the baseline you took in step 1. An alert is **resolved** only when the installed version no longer falls inside `vulnerable_version_range`. If it is still in range, the alert is **not** resolved. A gem that has disappeared from the lockfile entirely also resolves its alerts.

- Check every alerted gem this way, not just the one you upgraded.
- An upgrade will often resolve alerts for gems other than the one named, because transitive dependencies move or drop out with it. Include all of them.
- List every resolved alert under `## Vulnerabilities`, grouped by gem, and state the count.
- If an alerted gem was moved but is still vulnerable, or an alert relates to a gem this run deliberately did not touch, mention it under `## Notes` rather than claiming it as fixed. Name the parent that pins it, from the `deps why` output.
- Do not claim a count you have not verified against the version data. It is better to report fewer alerts accurately than to assume a whole gem's alerts are cleared.
- Dependabot will not close the alerts until the change is merged, so report them as resolved by this change rather than as already closed.
- Copy the GHSA id, CVE id and summary out of the alert JSON when writing them up. They are not the sort of thing to recall from memory, and a plausible-looking GHSA id that belongs to a different advisory is worse than no link at all.
- If the origin remote is not a GitHub repository, or the API call genuinely fails, omit the `## Vulnerabilities` section and state the specific error under `## Notes`.
- If the bundle includes `bundler-audit`, `bundle exec bundle-audit check --update` is a useful second opinion, but it never replaces the dependabot check above.

## 5. Verify

Run every check the project defines that is relevant to an upgrade, alongside `bundle install` itself. Find them rather than guessing, in this order:

- The CI workflow is the authoritative list, because it is the set of checks the project actually gates on. Read `.github/workflows/` and run what it runs.
- `denvig run` lists the actions the project exposes, and `bundle exec rake -T` lists the rake tasks. Either may name a check that CI drives some other way.
- `bin/` scripts and the `Rakefile` show the rest — typically some combination of `bundle exec rspec`, `bin/rails test`, `bundle exec rubocop` and `bundle exec standardrb`.
- If the project uses Sorbet (a `sorbet/` directory exists), run `bundle exec srb tc` as well.

Do not invent checks the project has no tooling for, and do not promote a tool to a gate that CI does not gate on. A linter can sit in the bundle with thousands of pre-existing offenses; running it proves nothing about your upgrade and reporting it as a failure is misleading. If you run one anyway, compare the offense count against the pre-upgrade tree and report the difference, not the total.

Record the outcome of each check. If any of them fail, fix the failure or report it clearly rather than claiming a clean run. A pre-existing failure that your upgrade did not cause still has to be reported as pre-existing — establish that by checking it against a clean tree before blaming the upgrade.

Some failures are environmental rather than real: a missing `.env`, an unmigrated or absent database, or a service the suite expects to be running. In a fresh worktree, `denvig run` often exposes an init action that links those files in. Fix the environment and re-run before reporting anything as a failure, and if you cannot, say the check was not run and why rather than reporting it as failing.

## 6. Commit and open a PR

Once you have completed the above steps for each gem you should summarize all the changes in the format below.

- If upgrading multiple dependencies then {{git_commit_message}} should be: `Update {{count}} (patch|minor) dependencies`
- If upgrading a single dependency then {{git_commit_message}} should be: `Upgrade {{gem}} from {{old version}} to {{new version}}`

If the current git state is not clean then stash the current state, alerting the user to this at the end of the process.
If the current branch is `main` then check the git remote `origin` to make the following choice:
- If `origin` is a github.com remote then create a new branch called `denvig/upgrade-{{count}}-{{type}}-gems`, or `denvig/upgrade-{{gem}}` when the user named a single gem.
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

- {{gem}} [{{old_version}} -> {{new_version}}]
  - [#{{alert_id}}]({{full_link_to_alert}}) [{{cve_id}}]({{full_link_to_advisory}}) ({{severity}}) {{alert_summary}}

## Package Updates

- {{gem}}: [{{old_version}} -> {{new_version}}]({{link_to_changelog_or_diff}})
  - {{summary_of_changes_from_changelog}}

## Code Changes

{{details_of_code_modifications_made}}

## Checks

{{list_of_checks_performed_after_upgrade}}
```

Notes on the template:

- The alert link is `https://github.com/{{owner}}/{{repo}}/security/dependabot/{{alert_id}}` and the advisory link is `https://github.com/advisories/{{ghsa_id}}`. Use the GHSA id as the link text when an alert has no CVE id.
- When a gem leaves the lockfile altogether, write the version pair as `[{{old_version}} -> removed]`.
- `## Package Updates` covers the gems you deliberately upgraded, with their changelogs. Transitive gems that merely moved as a side effect belong under `## Vulnerabilities`, not here, unless they required a code change.
- Use the single-dependency commit message form when the user named one gem, even if upgrading it moved many transitive gems.
- Omit the `## Vulnerabilities` section entirely when the upgrade resolved no alerts, and add a check line confirming that the alerts were read and none were affected.
- The vulnerability check line always takes the form `{{resolved}} of the {{open}} open alerts are resolved by this upgrade`, however large the remainder is. Alerts left open for reasons unrelated to this upgrade are normal and belong under `## Notes`, ideally saying which single upgrade would clear the most of them.

## Example Commit Message

### Patch

```markdown
Update 2 patch dependencies

## Package Updates

- puma: [6.4.2 -> 6.4.3](https://github.com/puma/puma/blob/master/History.md#643--2024-11-06)
  - Fix a rare race condition when a client disconnects during a chunked request
- nokogiri: [1.16.5 -> 1.16.7](https://github.com/sparklemotion/nokogiri/compare/v1.16.5...v1.16.7)
  - Update the vendored libxml2 to 2.12.9
  - Fix a memory leak in `Nokogiri::XML::Reader`

## Code Changes

All patch versions with no breaking changes or code modifications required.

## Checks

- ✅ Checked changelogs for breaking changes
- ✅ Checked dependabot alerts, none of the 3 open alerts are affected by these upgrades
- ✅ `bundle exec rubocop` passes after upgrade
- ✅ All tests pass after upgrade
- ✅ No code modifications were necessary
```

### Minor

```markdown
Update 2 minor dependencies

## Package Updates

- rubocop: [1.65.1 -> 1.66.1](https://github.com/rubocop/rubocop/compare/v1.65.1...v1.66.1)
  - New `Lint/UselessNumericOperation` cop, pending by default
  - `Style/RedundantParentheses` now handles one-line pattern matching
- sidekiq: [7.2.4 -> 7.3.2](https://github.com/sidekiq/sidekiq/blob/main/Changes.md)
  - Job execution now records the queue latency in the job hash
  - Web UI supports filtering the retry set by job class

## Code Changes

- Ran `bundle exec rubocop -A` to apply the autocorrectable changes from the new cops
- Enabled `Lint/UselessNumericOperation` in `.rubocop.yml` now that it ships

## Notes

- `Style/RedundantParentheses` flagged two pattern matches in `app/services/matcher.rb` that were previously ignored.

## Checks

- ✅ Checked changelogs for breaking changes
- ✅ Checked dependabot alerts, none of the 3 open alerts are affected by these upgrades
- ✅ `bundle exec rubocop` passes after upgrade
- ✅ All tests pass after upgrade
- ✅ Code modifications applied where necessary
```

### Major with vulnerabilities

```markdown
Upgrade rack from 2.2.9 to 3.1.8

## Vulnerabilities

Resolves 2 dependabot alerts.

- rack [2.2.9 -> 3.1.8]
  - [#14](https://github.com/marcqualie/example/security/dependabot/14) [CVE-2026-31022](https://github.com/advisories/GHSA-7g2v-jj9q-g3rq) (high) rack is vulnerable to denial of service via unbounded multipart parsing
  - [#15](https://github.com/marcqualie/example/security/dependabot/15) [GHSA-p4q6-4v9w-4pjr](https://github.com/advisories/GHSA-p4q6-4v9w-4pjr) (medium) rack::sendfile can be tricked into serving files outside the configured root

## Package Updates

- rack: [2.2.9 -> 3.1.8](https://github.com/rack/rack/blob/main/UPGRADE-GUIDE.md)
  - Response headers must now be lowercase, and the body is an enumerable rather than an array
  - `Rack::File` was removed in favour of `Rack::Files`
  - `Rack::Handler` moved out to the `rackup` gem

## Code Changes

- Replaced `Rack::File` with `Rack::Files` in `config/application.rb`
- Downcased the header keys returned by `Middleware::Cors` in `lib/middleware/cors.rb`
- Added the `rackup` gem to the `Gemfile`, which `rack` 3 no longer bundles

## Notes

- `some-legacy-gem` still declares `rack (~> 2.2)` so it was upgraded to 4.1.0 alongside this change to keep the bundle resolvable.
- Alert [#16](https://github.com/marcqualie/example/security/dependabot/16) is against the npm ecosystem and is out of scope for this upgrade.

## Checks

- ✅ Checked changelogs for breaking changes
- ✅ Checked dependabot alerts, 2 of the 3 open rubygems alerts are resolved by this upgrade
- ✅ `bundle exec rubocop` passes after upgrade
- ✅ All tests pass after upgrade
- ✅ Code modifications applied where necessary
```
