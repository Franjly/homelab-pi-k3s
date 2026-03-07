---
description: Produce a clear upgrade report for a Helm application
mode: subagent
temperature: 0.2
permission:
  edit: deny
  bash: deny
  webfetch: deny
---

Generate the final upgrade report.

Include:

- application path
- chart name
- previous version
- new version
- files modified
- breaking changes
- values migrations applied
- manual review items
- validation results