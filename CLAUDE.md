# Claude Code Instructions — homelab-pi-k3s

## Repo structure
This repo manages Helm-based app deployments on a k3s cluster using Flux or similar GitOps tooling.
Each app lives under a directory (e.g. `kubernetes/apps/<app-name>/`) and typically contains:
- `helmrelease.yaml` (or similar) — defines the chart version currently deployed
- `values.yaml` — custom overrides on top of the default Helm chart values

## Upgrade workflow
When asked to upgrade an app, follow these steps **in order**:

### Step 1 — Identify current version
- Find the app directory and read the chart version from `helmrelease.yaml` (or equivalent manifest)
- Note the chart name, repo URL, and current version

### Step 2 — Compare custom values vs chart defaults
- Fetch the default `values.yaml` from the Helm chart at the **current** version (use `helm show values <repo>/<chart> --version <current>`)
- Diff it against the local `values.yaml` to identify custom overrides
- Save this diff mentally — it must be preserved in the upgrade

### Step 3 — Find latest version
- Run `helm repo update` and `helm search repo <chart> --versions` to find the latest stable version
- Fetch the default `values.yaml` for the **latest** version
- Compare it against the current version's defaults to detect:
  - Renamed or removed keys → add a `# TODO: BREAKING CHANGE — <key> removed/renamed, see <changelog URL>` comment in values.yaml
  - New required values → incorporate them with sensible defaults or a TODO
  - Structural changes

### Step 4 — Apply the upgrade
- Update the chart version in `helmrelease.yaml`
- Merge the custom overrides into the new default structure, preserving all custom config
- Add `# TODO` comments for any breaking changes that need manual review

### Step 5 — Git branch + push
- Create a branch named `upgrade/<app-name>-<new-version>` (e.g. `upgrade/grafana-8.3.1`)
- Commit all changes with message: `chore: upgrade <app> from <old> to <new>`
- Push the branch

### Step 6 — Check release notes
- Look up the GitHub releases or changelog for the chart/app between the old and new version
- Identify: breaking changes, manual migration steps, deprecations, security advisories
- Summarize findings for the PR description

### Step 7 — Create PR
- Create a PR from the upgrade branch to `main`
- PR title: `chore: upgrade <app> from <old-version> to <new-version>`
- PR body must include:
  - Summary of what changed
  - List of preserved custom config
  - Any breaking changes or manual steps needed (with links to docs/release notes)
  - Any TODO items added to the code
- Output the PR URL at the end

## Tools available
- `helm` CLI
- `gh` CLI (GitHub CLI, already authenticated)
- `git`
- `curl` / `wget` for fetching changelogs if needed

## Important rules
- NEVER remove custom configuration without explicit user confirmation
- Always add `# TODO` comments rather than silently dropping unknown config
- Prefer stable/non-RC versions unless asked otherwise
- Do not touch other apps outside the one being upgraded