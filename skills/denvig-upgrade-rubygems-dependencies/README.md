# Upgrade rubygems dependencies

Uses the `denvig` cli tool to identify outdated gems and manually upgrade them by checking the changelogs for breaking changes and important notes. This process ensures that we have control over the upgrade process and can avoid any unexpected issues that may arise from automatic upgrades.

Every run cross references the result against the repository's dependabot alerts via the `gh` cli, so any CVEs fixed by the upgrade are listed in the commit message and pull request description. Alerts that remain open are called out too, along with the gem pinning them. In a polyglot repository only the `rubygems` alerts are counted, and the rest are noted so the totals are not misread.

Direct and transitive dependencies both follow the same process. A direct dependency is moved by editing the constraint in the `Gemfile` or gemspec, a transitive one by `bundle update --conservative`, and everything else — changelog review, vulnerability check, verification, commit and PR — is identical. Bare `bundle update` is never run.

## Installation

Ensure `denvig` is installed globally:

```bash
brew install denvig/tap/denvig
```

Then add the skill:

```bash
npx skills add marcqualie/agent-skills --skill denvig-upgrade-rubygems-dependencies
```

## Usage

```bash
claude "/denvig-upgrade-rubygems-dependencies"
claude "/denvig-upgrade-rubygems-dependencies minor"
```

Pass a gem name to upgrade just that one, whether it is declared in the `Gemfile` or pulled in transitively:

```bash
claude "/denvig-upgrade-rubygems-dependencies rails"
claude "/denvig-upgrade-rubygems-dependencies rack"
```
