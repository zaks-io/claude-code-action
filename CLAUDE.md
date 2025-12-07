# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with
code in this repository.

## Project Overview

This repository provides reusable GitHub Actions workflows for integrating
Claude Code into other repositories. It contains three workflows that wrap
`anthropics/claude-code-action@v1`:

- `claude.yml` - General-purpose agent for issue/PR comments
- `issue-triage.yml` - Automated issue triage with optional Linear MCP
  integration
- `code-review.yml` - Manual code review triggered by `/review` comment

## Commands

```bash
# Format YAML and Markdown files
npx prettier --write "**/*.{yml,yaml,md}"

# Check formatting (used in CI)
npx prettier --check "**/*.{yml,yaml,md}"

# Lint GitHub Actions workflows
actionlint

# Setup pre-commit hooks
pip install pre-commit
pre-commit install
```

## Architecture

All reusable workflows are in `.github/workflows/` and use `workflow_call`
triggers. Callers reference them as:

```yaml
uses: ORG/REPO/.github/workflows/claude.yml@v1
```

### Workflow Structure

Each workflow defines:

- `inputs` - Configurable parameters (allowed_tools, prompt, etc.)
- `secrets` - Required credentials (claude_code_oauth_token, github_token)
- `permissions` - GitHub token scopes needed

The `issue-triage.yml` workflow has conditional steps based on `enable_linear`
and `prompt` inputs to handle four combinations: with/without Linear,
with/without custom prompt.

The `code-review.yml` workflow includes additional `actions/github-script` steps
for check run management and emoji reactions.

## Versioning

Use semantic version tags (`v1`, `v1.0.0`) for releases. Callers should
reference `@v1` for automatic minor/patch updates.
