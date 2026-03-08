---
description: Compare Helm values across chart versions and propose safe migrations
mode: subagent
temperature: 0
permission:
  edit: deny
  bash:
    "cat *": allow
    "grep *": allow
    "helm show *": allow
    "python *": allow
    "python3 *": allow
    "helm repo *": allow
    "true *": allow
    "helm search *": allow
    "mktemp *": allow
    "helm pull *": allow
    "ls *": allow
    "rg *": allow
    "gh api *": allow
    "gh release list *": allow
    "printf *": allow
    "rg *": allow
    "/tmp/*": allow
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