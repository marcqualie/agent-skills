# Upgrade npm dependencies

Uses the `denvig` cli tool to identify outdated dependencies and manually upgrade them by checking the changelogs for breaking changes and important notes. This process ensures that we have control over the upgrade process and can avoid any unexpected issues that may arise from automatic upgrades.

## Installation

Ensure `denvig` is installed globally:

```bash
brew install denvig/tap/denvig`
```

Then add the skill:

```bash
npx skills add marcqualie/agent-skills --skill denvig-upgrade-npm-dependencies
```

## Usage

```bash
claude "/denvig-upgrade-npm-dependencies"
```

