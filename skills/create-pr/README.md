# Create a Pull Request using gh CLI

Summarize the work on the current branch and create a draft pull request using the GitHub CLI. The pull request will be assigned to @me and the description will be a summary of the changes on this branch. After creation, the link to the PR will be shown and the PR will be opened in the browser.

## Installation

```bash
npx skills add marcqualie/agent-skills --skill create-pr
```

## Usage

```
/create-pr
```

or with custom instructions:

```
/create-pr remove all of the console.logs before summarising and committing.
```
