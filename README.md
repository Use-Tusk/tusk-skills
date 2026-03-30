# Tusk Skills

Agent skills for [Tusk](https://usetusk.ai) — add AI-powered testing workflows to your editor with a single command.

## Overview

Tusk Skills are portable AI agent instructions that integrate Tusk's automated testing capabilities directly into your development workflow. Each skill teaches your AI coding assistant (Cursor, Claude Code, etc.) how to perform a specific Tusk task — like analyzing API test deviations, debugging regressions, or triaging failures — without leaving your editor.

This repository is the official skill registry for Tusk.

## Install

Install all skills:

```bash
npx skills add Use-Tusk/tusk-skills
```

Install a specific skill:

```bash
npx skills add Use-Tusk/tusk-skills --skill tusk-drift-analyze
```

Install to specific agents:

```bash
npx skills add Use-Tusk/tusk-skills -a claude-code -a cursor
```

## Quick Start

1. Install the skill:

   ```bash
   npx skills add Use-Tusk/tusk-skills
   ```

2. Open your AI agent (Cursor, Claude Code, etc.) in your project

3. Ask your agent to analyze drift deviations:

   ``` bash
   Run my Tusk Drift tests and analyze the deviations.
   ```

## Available Skills

| Skill | Description |
|-------|-------------|
| **[tusk-drift-analyze](skills/tusk-drift-analyze/SKILL.md)** | Analyze Tusk Drift API test deviations locally — classifies deviations as intended, unintended, or unrelated, and optionally fixes regressions. |

## Related Links

- [Tusk CLI](https://github.com/Use-Tusk/tusk-cli)
- [Tusk Drift Python SDK](https://github.com/Use-Tusk/drift-python-sdk)
- [Tusk Drift Node.js SDK](https://github.com/Use-Tusk/drift-node-sdk)
- [Tusk Documentation](https://docs.usetusk.ai)

## License

Apache 2.0 — see [LICENSE](LICENSE) for details.
