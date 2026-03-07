---
description: Compare Helm values across chart versions and propose safe migrations
mode: subagent
temperature: 0
permission:
  edit: deny
  bash:
    "cat *": allow
    "grep *": allow
    "*": ask
  webfetch: allow
---

Compare:

- the user's current `values.yaml`
- default values of the current chart version
- default values of the target chart version

Identify:

- unchanged keys
- renamed keys
- removed keys
- schema changes
- new relevant configuration

Propose minimal changes needed to preserve behavior.