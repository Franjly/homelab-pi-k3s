## Purpose
This repository is a YAML-first Kubernetes homelab repo built around Helm wrapper charts, Kustomize, and Argo CD config-management plugins.
Agents should optimize for safe manifest edits, minimal git noise, and parity with the render flow defined in `platform/argocd/values.yaml`.
## Rule Sources
- No prior root `AGENTS.md` was present.
- No `.cursorrules` file was found.
- No files were found under `.cursor/rules/`.
- No `.github/copilot-instructions.md` file was found.
- If any of those files appear later, treat them as higher-priority supplements.
- Also check for directory-local `README.md` files before changing behavior in that subtree.
## Repo Layout
- `apps/`: application workloads.
- `platform/`: platform services like Argo CD, Vault, Gitea, and Trivy.
- `system/`: cluster infrastructure like ingress, cert-manager, storage, and monitoring.
- Most Helm-backed directories contain `Chart.yaml`, `values.yaml`, `kustomization.yaml`, and a generated `resources/all.yaml`.
- Some directories are pure Kustomize or raw manifests, such as `apps/pod-info/` and `apps/external-endpoints/`.
## Render Model
The common pattern in this repo is:
1. `helm dependency build`
2. `helm template ... > resources/all.yaml`
3. `kustomize build`
That sequence is explicitly encoded in the Argo CD CMP definitions in `platform/argocd/values.yaml` and `platform/argocd/values-no-metrics.yaml`.
Some directories may still document local exceptions; for example, `system/calico/README.md` notes a direct Helm deployment caveat.
## Tooling
- Expected tools: `helm`, `kustomize` or `kubectl kustomize`, and `kubectl`.
- Helpful optional tools: `yamllint`, `yq`, and `kubeconform`.
- No repo-level `Makefile`, `Taskfile`, `package.json`, `pyproject.toml`, GitHub Actions workflow, or pre-commit config exists in the current tree.
## Build Commands
There is no single repo-wide build command; build the directory you changed.
### Build one Helm-backed directory without modifying tracked files
```bash
tmpdir=$(mktemp -d) && \
cp -R apps/arr/overseerr/. "$tmpdir" && \
helm dependency build "$tmpdir" && \
helm template overseerr "$tmpdir" --namespace arr --include-crds -f "$tmpdir/values.yaml" > "$tmpdir/resources/all.yaml" && \
kustomize build "$tmpdir"
```
### Build one Helm-backed directory and refresh `resources/all.yaml`
```bash
helm dependency build apps/arr/overseerr && \
helm template overseerr apps/arr/overseerr --namespace arr --include-crds -f apps/arr/overseerr/values.yaml > apps/arr/overseerr/resources/all.yaml && \
kustomize build apps/arr/overseerr
```
### Build one pure Kustomize directory
```bash
kustomize build apps/pod-info
```
### Fallback if `kustomize` is unavailable
```bash
kubectl kustomize apps/pod-info
```

### Build policy for agents
- Prefer validating one changed directory, not the whole repo.
- Use a temp copy for Helm-backed validation unless you intentionally want a regenerated `resources/all.yaml` diff.
- If you do refresh `resources/all.yaml`, make sure the matching `kustomize build` still passes.
## Lint Commands
There is no dedicated repo-wide linter config, so lint by manifest type.
### Lint one Helm-backed directory
```bash
helm lint apps/arr/overseerr
```
If dependencies are missing:
```bash
helm dependency build apps/arr/overseerr && helm lint apps/arr/overseerr
```
### Validate a rendered manifest stream with the Kubernetes client schema
```bash
kustomize build apps/pod-info | kubectl apply --dry-run=client -f -
```
### Optional lint/schema checks
```bash
yamllint apps/pod-info
yamllint apps/arr/overseerr/values.yaml
kustomize build apps/pod-info | kubeconform -strict -summary -ignore-missing-schemas
```
## Test Commands
There is no conventional unit or integration test harness in this repo. In practice, "tests" here mean rendering and validating manifests.
### Fast test for one pure-manifest app
```bash
kustomize build apps/pod-info | kubectl apply --dry-run=client -f -
```
### Fast test for one Helm-backed app
```bash
tmpdir=$(mktemp -d) && \
cp -R apps/arr/overseerr/. "$tmpdir" && \
helm dependency build "$tmpdir" && \
helm template overseerr "$tmpdir" --namespace arr --include-crds -f "$tmpdir/values.yaml" > "$tmpdir/resources/all.yaml" && \
kustomize build "$tmpdir" | kubectl apply --dry-run=client -f -
```
### Test a single changed manifest file
```bash
kubectl apply --dry-run=client -f apps/external-endpoints/resources/pi-hole-0.yaml
```
### Closest equivalent to "run a single test"
- single Helm unit: one directory like `apps/arr/overseerr/`
- single Kustomize unit: one directory like `apps/pod-info/`
- single raw-manifest unit: one file like `apps/external-endpoints/resources/pi-hole-0.yaml`

### Minimum validation expectation
- values-only change in a Helm-backed dir: `helm lint` plus the render/build chain.
- raw manifest change: `kubectl apply --dry-run=client -f` on the file plus parent `kustomize build`.
- Kustomize-only change: `kustomize build` on that directory.

## Commands By Change Type
- edited `values.yaml` in a Helm-backed directory: run `helm lint`, then the full `helm template -> resources/all.yaml -> kustomize build` chain.
- edited `kustomization.yaml`: run `kustomize build` on that directory.
- edited raw manifests under `resources/`: run `kubectl apply --dry-run=client -f` on the file, then `kustomize build` for the parent directory.
- edited Vault placeholders like `<path:...#...>`: validate rendering only; do not replace placeholders with concrete secrets.

## Agent Operating Rules
- Make the smallest targeted change that solves the task.
- Preserve surrounding structure and style; do not opportunistically rewrite unrelated YAML.
- Do not add new top-level values keys when an existing subtree already fits the change.
- Do not remove upstream comments from large values files unless they are clearly wrong.
- Do not replace placeholder secrets with live credentials, even temporarily.
- Treat generated output as derived state: only refresh it when your task requires it.
## Code Style
This repo is mostly YAML, Kustomize, and Helm values, so style rules are manifest-focused.
### Naming
- Use lowercase kebab-case for directories, file names, chart names, namespaces, resource names, TLS secrets, and most hostnames.
- Keep names aligned across `Chart.yaml`, `kustomization.yaml`, ingress hosts, service names, and secret names when possible.
- Preserve existing namespace patterns such as `arr`, `argocd`, `external-endpoints`, and directory-specific namespaces.
### Formatting
- Use spaces, not tabs.
- Follow the existing indentation in the touched file; most files use two-space nesting.
- Keep list indentation consistent within a file, even if older files have minor quirks.
- Avoid formatting-only diffs.
- Do not broadly reformat large vendor-style `values.yaml` files.
- In raw manifest files, keep logical resources separated with `---`.
### Imports, Types, And Templating
- Traditional language imports are usually not applicable in this repo.
- Helm references like `{{ .Values... }}` and `{{ .Release.Name }}` are the closest thing to typed references; keep them explicit and simple.
- Prefer existing chart value structures over inventing new top-level keys.
- Preserve scalar intent: if a field is already a boolean, integer, or string, keep that type.
- Quote strings when the surrounding file already quotes similar values, especially for host names, annotations, templated strings, and values YAML may misread.
### Chart And Kustomize Conventions
- Wrapper `Chart.yaml` files typically pin one dependency and keep `version: 0.0.0`.
- Do not rename wrapper charts casually; directory name, chart name, and render path should stay aligned.
- Helm-backed `kustomization.yaml` files usually point at `./resources/all.yaml`.
- If you regenerate `resources/all.yaml`, make sure `kustomize build` still succeeds.
- Keep `kustomization.yaml` files minimal and declarative.
- Use relative paths like `./resources/...` to match current style.
### Kubernetes Manifest Conventions
- Prefer explicit metadata names and stable selectors.
- Do not change selectors unless the replacement is intentional and safe.
- Prefer `ingressClassName` over legacy ingress-class annotations when the file already uses it.
- Reuse existing annotation patterns like `cert-manager.io/cluster-issuer` and `hajimari.io/*` for user-facing services.
- Reuse existing storage classes such as `longhorn`, `nfs-media`, `nfs-downloads`, `nfs-prometheus`, and `nfs-roms` instead of inventing new names casually.
### Error Handling And Secrets
- Do not hardcode secrets, tokens, API keys, usernames, or passwords.
- Preserve Vault placeholders like `<path:pi-k3s/data/...#key>`.
- When adding sensitive config, prefer existing patterns such as `secretKeyRef`, `stringData`, and Vault placeholders.
- Treat render failures as real errors; do not ignore broken templates or missing fields.
- For cluster-wide changes, prefer the smallest targeted diff.
### Comments And Docs
- Keep useful upstream comments in large values files unless they become misleading.
- Add comments only when the intent is not obvious from the manifest.
- If a touched directory has a local README, update it when operational behavior changes.

## Commit And PR Dos/Don'ts
- Do keep commits narrowly scoped to the changed app or system.
- Do mention the operational reason for the change, not just the file names.
- Do include validation commands run for the touched directory in your handoff or PR body.
- Don't mix unrelated chart upgrades, refactors, and manifest fixes in one change.
- Don't commit secrets, rendered credentials, kubeconfig data, or substituted Vault values.
- Don't regenerate `resources/all.yaml` for unrelated directories.
## Agent Workflow
- First identify whether the target is Helm-backed, pure Kustomize, or raw manifests.
- After editing, run the narrowest validation command that matches the change.
- Prefer single-directory validation over repo-wide sweeps.
- Use a temp copy for Helm validation unless you intentionally want regenerated output in git.
- Never replace secret placeholders with live values just to make local validation pass.
- Check for a local `README.md` before changing deployment behavior in a specific directory.



## Helm Chart Upgrade Automation

This repository supports AI-assisted Helm chart upgrade workflows.

Agents may be asked not only to analyze upgrades but also to **apply upgrade changes directly in the repository** so that the user can review the resulting diff.

The goal is to automate the mechanical parts of Helm upgrades while preserving user intent and minimizing risk.

### Upgrade Workflow

When performing an upgrade task for an application, agents should:

1. Detect the currently configured chart dependency in `Chart.yaml`
2. Identify the upstream Helm repository
3. Determine the latest available chart version
4. Review release notes and changelogs between current and target versions
5. Detect breaking changes or configuration schema changes
6. Compare:
   - the repository's current `values.yaml`
   - default values for the current chart version
   - default values for the target chart version
7. Apply the **minimal changes required** to support the new version
8. Preserve all existing user-specific configuration whenever possible

### Allowed Changes

During an upgrade task, agents may modify repository files.

Typical modifications include:

- updating the chart dependency version in `Chart.yaml`
- applying minimal migrations to `values.yaml`
- regenerating derived manifests such as `resources/all.yaml` when required by the repository workflow

Changes must remain **small, targeted, and reviewable**.

### Upgrade Safety Principles

Agents must follow these rules:

- preserve existing user configuration whenever possible
- prefer minimal diffs instead of rewriting entire files
- do not modify unrelated applications
- avoid reformatting YAML unnecessarily
- avoid removing comments unless required by the edit
- do not replace secret placeholders with real credentials

If a migration cannot be determined with high confidence:

- apply only safe changes
- leave clear notes for manual review

### High-Priority Configuration

The following settings usually represent cluster-specific decisions and must be preserved whenever possible:

- ingress configuration
- TLS configuration
- annotations and labels
- `service.type`
- `loadBalancerIP`
- persistence configuration
- storage classes
- image repository/tag overrides
- environment variables
- secret references
- resource limits/requests
- `nodeSelector`
- `tolerations`
- `affinity`
- security context settings

### Validation After Upgrade

After applying upgrade changes to a Helm-backed application, agents should validate the result using the repository's render workflow:

1. `helm dependency build`
2. `helm lint`
3. `helm template ... > resources/all.yaml` (when applicable)
4. `kustomize build`

If validation fails, agents should report the failure clearly and indicate which manual follow-up may be required.

### Upgrade Result Reporting

After performing an upgrade, agents should report:

- app path
- chart name
- previous version
- new version
- files modified
- detected breaking changes
- applied values migrations
- manual review items
- validation results

### Future Pull Request Automation

This repository may later support automatic Pull Request creation for upgrades.

When PR automation is enabled, agents should:

- create one branch per upgraded application
- commit only the relevant changes
- generate a clear commit message
- generate a PR description including upgrade details and review notes



## Repository-Wide Upgrade Scanning

This repository may also use AI agents to scan all Helm-backed applications and identify available chart upgrades.

For repository-wide upgrade scans, agents should:

1. discover all application directories that contain `Chart.yaml`
2. extract chart name, repository, and current version
3. check for newer upstream versions
4. detect deprecations, maintainer changes, and migration scenarios
5. classify each app as:
   - up to date
   - upgrade available
   - breaking upgrade
   - deprecated chart
   - migration required

Repository-wide scans should not modify files unless explicitly requested.

The expected output is a prioritized report that helps decide which applications are the best upgrade candidates.



## Batch Upgrade Planning

This repository may also use AI agents to build a prioritized upgrade plan across all Helm-backed applications.

For repository-wide upgrade planning, agents should:

1. discover all Helm-backed applications
2. determine which ones have newer chart versions available
3. classify each upgrade as:
   - safe upgrade candidate
   - manual-review upgrade
   - migration-required upgrade
   - up to date
4. assign a risk level to each application
5. recommend an execution order

The goal of batch upgrade planning is to help the user decide which upgrades should be performed first.

Batch upgrade planning should not modify repository files unless explicitly requested.



## Pull Request Preparation

This repository may also use AI agents to prepare Pull Request metadata for app upgrades that have already been applied.

For PR preparation tasks, agents should:

1. inspect the current git diff
2. confirm which files were changed for the target application
3. infer the previous and new chart versions when possible
4. generate:
   - a branch name
   - a commit message
   - a PR title
   - a PR description
5. clearly indicate whether the diff appears cleanly scoped to one application

PR preparation should not automatically create commits, push branches, or open Pull Requests unless explicitly enabled in the future.



## Pull Request Creation

This repository may also use AI agents to create Pull Requests for single-application upgrades that have already been applied and reviewed for scope.

For PR creation tasks, agents should:

1. inspect the current git diff
2. verify that the changes are cleanly scoped to a single application
3. create a dedicated branch
4. stage only the relevant files
5. create a focused commit
6. generate or reuse:
   - a branch name
   - a commit message
   - a PR title
   - a PR description
7. create the Pull Request when repository tooling or integration supports it

Agents must not:

- include unrelated changes
- force-push
- rewrite history
- merge automatically
- create PRs for messy or mixed diffs

If automatic PR creation is not supported in the environment, agents should still prepare the branch, commit, and PR metadata when possible, and report what remains to be done manually.



## Batch Upgrade Execution

This repository may also use AI agents to execute a small batch of safe application upgrades.

For batch upgrade execution, agents should:

1. start from a repository-wide upgrade plan
2. select only safe upgrade candidates by default
3. keep the default batch size very small
4. execute each application upgrade independently
5. validate each upgraded application
6. create one branch, one commit, and one Pull Request per application when supported

Batch execution must be conservative.

By default, agents should avoid:

- deprecated charts
- migration-required upgrades
- major breaking upgrades
- upgrades requiring unclear values migrations

Agents must prioritize safety, isolation, and reviewability over execution speed.



## Post-Upgrade Review

This repository may also use AI agents to review an already-applied application upgrade before commit or Pull Request creation.

For post-upgrade review tasks, agents should:

1. inspect the current diff for the target application
2. verify that the changed files are expected for a normal upgrade
3. check whether sensitive user configuration changed
4. identify suspicious or low-confidence migrations
5. classify the result as:
   - ready for PR creation
   - ready with manual review
   - not ready for PR creation

Post-upgrade review should be conservative and should help prevent unsafe automated Pull Requests.