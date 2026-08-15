# Upgrade npm dependencies

Uses the `denvig` cli tool to identify outdated dependencies and manually upgrade them by checking the changelogs for breaking changes and important notes. This process ensures that we have control over the upgrade process and can avoid any unexpected issues that may arise from automatic upgrades.

Every run cross references the result against the repository's dependabot alerts via the `gh` cli, so any CVEs fixed by the upgrade are listed in the commit message and pull request description. Alerts that remain open are called out too, along with the package pinning them.

Direct and transitive dependencies both follow the same process. A direct dependency is moved by editing `package.json`, a transitive one by `pnpm -r upgrade`, and everything else — changelog review, vulnerability check, verification, commit and PR — is identical.

## Installation

Ensure `denvig` is installed globally:

```bash
brew install denvig/tap/denvig
```

Then add the skill:

```bash
npx skills add marcqualie/agent-skills --skill denvig-upgrade-npm-dependencies
```

## Usage

```bash
claude "/denvig-upgrade-npm-dependencies"
claude "/denvig-upgrade-npm-dependencies minor"
```

Pass a package name to upgrade just that one, whether it is a direct dependency or buried in the tree:

```bash
claude "/denvig-upgrade-npm-dependencies wrangler"
claude "/denvig-upgrade-npm-dependencies undici"
```
