---
name: audit
description: Run npm audit and optionally fix vulnerabilities
disable-model-invocation: true
allowed-tools: Bash(npm audit:*), Bash(npm audit fix:*)
---

## Context

- Current audit report: !`npm audit 2>&1`

## Your task

Review the audit report above and summarize the vulnerabilities found (critical, high, moderate, low counts).

If there are vulnerabilities that can be automatically fixed, run `npm audit fix` to resolve them. If there are breaking-change fixes available (requiring `npm audit fix --force`), list them separately and ask the user before proceeding — do not run `--force` automatically.

After fixing, run `npm audit` again and report the remaining issues.
