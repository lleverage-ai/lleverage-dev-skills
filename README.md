# Lleverage Dev Skills

Lleverage agent skills for Claude Code development workflows.

## Installation

Add the marketplace to Claude Code:

```bash
claude plugin marketplace add lleverage-ai/lleverage-dev-skills
```

Install a plugin:

```bash
claude plugin install code-review@lleverage-dev-skills
```

## Plugins

### code-review

Automated code review for pull requests using multiple specialized agents with confidence-based scoring.

```bash
/code-review https://github.com/owner/repo/pull/123
```

## Development

To test plugins locally:

```bash
claude plugin install /path/to/agent-skills/plugins/code-review --scope user
```
