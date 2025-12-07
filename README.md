# Claude Code Reusable Workflows

Reusable GitHub Actions workflows for integrating
[Claude Code](https://github.com/anthropics/claude-code-action) into your
repositories.

## Workflows

| Workflow           | Description                                             |
| ------------------ | ------------------------------------------------------- |
| `claude.yml`       | General-purpose Claude Code agent for issue/PR comments |
| `issue-triage.yml` | Automated issue triage with Linear integration          |
| `code-review.yml`  | Code review (manual `/review` or automatic on PR open)  |

## Prerequisites

### Required Secrets

Set these secrets in your repository or organization:

| Secret                    | Required | Description                               |
| ------------------------- | -------- | ----------------------------------------- |
| `CLAUDE_CODE_OAUTH_TOKEN` | Yes      | Claude Code OAuth token                   |
| `LINEAR_API_KEY`          | No       | Required only if using Linear integration |

> **Note:** `GITHUB_TOKEN` is automatically available in all workflows - no need
> to pass it.

### Repository Access (Private Repos)

If this repository is private, enable access for other repositories:

1. Go to **Settings** > **Actions** > **General**
2. Under "Access", select "Accessible from repositories in the organization"
3. Choose which repositories can access these workflows

## Quick Start

### Basic Setup

Create `.github/workflows/claude.yml` in your repository:

```yaml
name: Claude Code

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned, labeled]
  pull_request_review:
    types: [submitted]

jobs:
  claude:
    uses: zaks-io/claude-code-action/.github/workflows/claude.yml@main
    secrets:
      claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

### With Issue Triage

```yaml
name: Claude Code

on:
  issues:
    types: [opened]

jobs:
  triage:
    uses: zaks-io/claude-code-action/.github/workflows/issue-triage.yml@main
    with:
      linear_team_prefix: 'PROJ'
    secrets:
      claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
      linear_api_token: ${{ secrets.LINEAR_API_KEY }}
```

### With Manual Code Review

Triggered by `/review` comment on a PR:

```yaml
name: Claude Code

on:
  issue_comment:
    types: [created]

jobs:
  review:
    if: |
      github.event.issue.pull_request &&
      contains(github.event.comment.body, '/review')
    uses: zaks-io/claude-code-action/.github/workflows/code-review.yml@main
    secrets:
      claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

### With Automatic Code Review

Triggered automatically on PR open/ready (use in CI pipeline):

```yaml
name: CI

on:
  pull_request:
    types: [opened, ready_for_review]

jobs:
  # ... your other CI jobs ...

  code-review:
    needs: [lint, test, build]
    if: |
      github.event.pull_request.draft == false &&
      github.event.pull_request.auto_merge == null
    uses: zaks-io/claude-code-action/.github/workflows/code-review.yml@main
    with:
      trigger_type: auto
    secrets:
      claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

## Workflow Reference

### claude.yml

General-purpose Claude Code agent for responding to comments.

#### Inputs

| Input           | Type   | Default                                                                                   | Description                           |
| --------------- | ------ | ----------------------------------------------------------------------------------------- | ------------------------------------- |
| `allowed_tools` | string | `Edit,Write,Read,Glob,Grep,LS,Bash(git:*),Bash(bun:*),Bash(npm:*),Bash(npx:*),Bash(gh:*)` | Comma-separated list of allowed tools |
| `prompt`        | string | `''`                                                                                      | Custom prompt to append               |

#### Secrets

| Secret                    | Required | Description             |
| ------------------------- | -------- | ----------------------- |
| `claude_code_oauth_token` | Yes      | Claude Code OAuth token |

#### Permissions

```yaml
permissions:
  contents: write
  pull-requests: write
  issues: write
  id-token: write
  actions: read
```

---

### issue-triage.yml

Automated issue triage with Linear integration.

#### Inputs

| Input                | Type   | Default                                                              | Description                                      |
| -------------------- | ------ | -------------------------------------------------------------------- | ------------------------------------------------ |
| `linear_team_prefix` | string | `''`                                                                 | Linear team prefix (e.g., `PROJ` for `PROJ-123`) |
| `allowed_tools`      | string | `Read,Grep,Glob,LS,Bash(gh issue:*),Bash(gh label:*),mcp__linear__*` | Comma-separated list of allowed tools            |
| `prompt`             | string | `''`                                                                 | Custom prompt (overrides default)                |

#### Secrets

| Secret                    | Required | Description             |
| ------------------------- | -------- | ----------------------- |
| `claude_code_oauth_token` | Yes      | Claude Code OAuth token |
| `linear_api_token`        | Yes      | Linear API token        |

#### Permissions

```yaml
permissions:
  contents: read
  pull-requests: read
  issues: write
  id-token: write
```

#### Default Behavior

When no custom `prompt` is provided, the workflow:

1. Analyzes the issue title and description
2. Identifies the issue type (bug, feature, enhancement, question)
3. Researches related code in the repository
4. Asks clarifying questions if needed
5. Applies appropriate labels
6. Creates/links Linear tickets
7. Posts an analysis summary

---

### code-review.yml

Code review workflow supporting both manual (`/review` comment) and automatic
(PR open) triggers.

#### Inputs

| Input           | Type   | Default                                                                                                                 | Description                                             |
| --------------- | ------ | ----------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| `trigger_type`  | string | `comment`                                                                                                               | `comment` for /review command, `auto` for PR open/ready |
| `allowed_tools` | string | `Read,Grep,Glob,LS,Bash(gh pr comment:*),Bash(gh pr diff:*),Bash(gh pr view:*),Bash(gh run list:*),Bash(gh run view:*)` | Comma-separated list of allowed tools                   |
| `prompt`        | string | `''`                                                                                                                    | Custom prompt (overrides default)                       |

#### Secrets

| Secret                    | Required | Description             |
| ------------------------- | -------- | ----------------------- |
| `claude_code_oauth_token` | Yes      | Claude Code OAuth token |

#### Permissions

```yaml
permissions:
  contents: read
  pull-requests: write
  checks: write
  id-token: write
  actions: read
```

#### Default Behavior

When no custom `prompt` is provided, the review covers:

1. **Code Quality** - Clean code, error handling, readability, KISS/DRY
2. **Security** - Vulnerabilities, input sanitization, auth logic, secrets
3. **Performance** - Bottlenecks, query efficiency, memory leaks
4. **Testing** - Coverage, test quality, edge cases
5. **Documentation** - Code docs, README updates, API docs

**When `trigger_type: comment` (default):**

- Creates a "Code Review" check run
- Reacts with 👀 emoji on start
- Gathers PR context (workflow runs, check runs)
- Reacts with 🚀 (success) or 😕 (failure)

**When `trigger_type: auto`:**

- Runs a streamlined review without comment reactions or check runs

## Complete Example

A full workflow using all three reusable workflows:

```yaml
name: Claude Code

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, labeled]
  pull_request_review:
    types: [submitted]

jobs:
  # General Claude agent for comments
  claude:
    if: |
      !contains(github.event.comment.body, '/review') &&
      github.event_name != 'issues'
    uses: zaks-io/claude-code-action/.github/workflows/claude.yml@main
    secrets:
      claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}

  # Issue triage on new issues
  triage:
    if: github.event_name == 'issues' && github.event.action == 'opened'
    uses: zaks-io/claude-code-action/.github/workflows/issue-triage.yml@main
    with:
      linear_team_prefix: 'PROJ'
    secrets:
      claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
      linear_api_token: ${{ secrets.LINEAR_API_KEY }}

  # Manual code review via /review comment
  review:
    if: |
      github.event_name == 'issue_comment' &&
      github.event.issue.pull_request &&
      contains(github.event.comment.body, '/review')
    uses: zaks-io/claude-code-action/.github/workflows/code-review.yml@main
    secrets:
      claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

## Custom Prompts

All workflows support custom prompts to override default behavior:

```yaml
jobs:
  triage:
    uses: zaks-io/claude-code-action/.github/workflows/issue-triage.yml@main
    with:
      prompt: |
        You are a helpful assistant. Analyze this issue and:
        1. Categorize it as bug, feature, or question
        2. Suggest relevant labels
        3. Estimate complexity (S/M/L)
    secrets:
      claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
      linear_api_token: ${{ secrets.LINEAR_API_KEY }}
```

## Development

### Setup

Install pre-commit hooks:

```bash
pip install pre-commit
pre-commit install
```

### Formatting

Format files manually:

```bash
npx prettier --write "**/*.{yml,yaml,md}"
```

### Linting

Run actionlint locally:

```bash
actionlint
```

## License

MIT
