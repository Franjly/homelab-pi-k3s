---
description: Inspect an application directory and extract Helm chart metadata
mode: subagent
temperature: 0
permission:
  edit: deny
  bash:
    "find *": allow
    "grep *": allow
    "ls *": allow
    "cat *": allow
    "*": ask
  webfetch: deny
---

Inspect the repository structure for a specific application.

Extract:

- application path
- presence of `Chart.yaml`
- chart dependency name
- chart dependency version
- chart dependency repository
- existence of `values.yaml`
- presence of `kustomization.yaml`
- additional directories such as `resources/` or `configs/`

Summarize important user overrides from `values.yaml`.