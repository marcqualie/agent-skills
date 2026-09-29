# Upgrade GitHub Actions

Uses the `denvig` cli tool to identify outdated GitHub Actions and manually upgrade them by checking the changelogs for breaking changes and important notes. This process ensures that we have control over the upgrade process and can avoid any unexpected issues that may arise from automatic upgrades.

Every action is pinned to a full commit SHA with the version in a trailing comment, because tags can be moved to point at different code and a SHA cannot:

```yaml
- uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
```

Actions that are already up to date but still referenced by tag are pinned at their current version during the same run. Stale version comments on existing pins are detected and corrected, since `denvig` resolves the version from the SHA rather than trusting the comment.

## Installation

Ensure `denvig` is installed globally:

```bash
brew install denvig/tap/denvig
```

Then add the skill:

```bash
npx skills add marcqualie/agent-skills --skill denvig-upgrade-github-actions
```

## Usage

Upgrade every outdated action and pin everything:

```bash
claude "/denvig-upgrade-github-actions"
```

Pin every action at its current version without upgrading anything:

```bash
claude "/denvig-upgrade-github-actions pin"
```

Pass an action name to upgrade and pin just that one:

```bash
claude "/denvig-upgrade-github-actions actions/checkout"
```
