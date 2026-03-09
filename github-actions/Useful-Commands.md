# GitHub Actions: Useful Commands

> **The Anvaya:** *`gh`  is the CLI bridge between your terminal and GitHub Actions — manage workflows, secrets, and runs without opening a browser.*

---

## 🔎 Inspecting Workflows & Runs

**List all workflows in the repo**

* **Why:** See every workflow file and its ID without browsing the UI.
* **Command:**

```bash
gh workflow list
```

**List recent runs for a specific workflow**

* **Why:** Check run history and status for a specific pipeline.
* **Command:**

```bash
gh run list --workflow ci.yml --limit 10
```

**Watch a live run in the terminal**

* **Why:** Tail live logs without opening the browser — works from SSH sessions.
* **Command:**

```bash
gh run watch <run-id>
# Or: watch the most recent run
gh run watch $(gh run list --workflow ci.yml --limit 1 --json databaseId -q '.[0].databaseId')
```

**View full logs of a run**

* **Why:** Retrieve failure logs for debugging without clicking through the UI.
* **Command:**

```bash
gh run view <run-id> --log
gh run view <run-id> --log-failed   # Only failed steps
```

---

## ▶️ Triggering & Managing Runs

**Manually trigger a `workflow_dispatch` workflow**

* **Why:** Run a workflow from the CLI with inputs, skipping the browser UI.
* **Command:**

```bash
gh workflow run deploy.yml --field environment=staging
gh workflow run deploy.yml -f environment=prod -f version=1.2.3
```

**Re-run a failed workflow**

* **Why:** Retry a flaky test or transient cloud error without a new commit.
* **Command:**

```bash
gh run rerun <run-id>
gh run rerun <run-id> --failed-only   # Only re-run failed jobs
```

**Cancel a running workflow**

* **Why:** Stop an accidental deploy or a long-running stale run.
* **Command:**

```bash
gh run cancel <run-id>
```

**Download artifacts from a run**

* **Why:** Retrieve build outputs, test reports, or binaries produced by a workflow.
* **Command:**

```bash
gh run download <run-id>                          # All artifacts
gh run download <run-id> -n app-binary-<sha>      # Specific artifact by name
gh run download <run-id> --dir ./build-output     # Into a specific directory
```

---

## 🔐 Secrets & Variables

**Set a repository secret**

* **Why:** Create or update a secret without copy-pasting into the GitHub UI.
* **Command:**

```bash
gh secret set MY_API_KEY                          # Prompts for value interactively
gh secret set MY_API_KEY --body "value"           # Inline value (avoid in shell history)
gh secret set MY_API_KEY < ./secret.txt           # From file (best for certs/keys)
```

**Set an environment-scoped secret**

* **Why:** Secrets scoped to `prod` environment are isolated from other jobs.
* **Command:**

```bash
gh secret set DEPLOY_KEY --env prod < ./prod_key.pem
```

**List all secrets (names only — values are never shown)**

* **Why:** Audit what secrets exist without accessing their values.
* **Command:**

```bash
gh secret list
gh secret list --env prod
```

**Delete a secret**

* **Why:** Remove stale or compromised credentials.
* **Command:**

```bash
gh secret delete OLD_API_KEY
```

**Set a repository variable (non-secret config)**

* **Why:** Store non-sensitive config (regions, image names) accessible via `${{ vars.NAME }}`.
* **Command:**

```bash
gh variable set AWS_REGION --body "us-east-1"
gh variable set IMAGE_REPO --body "ghcr.io/myorg/myapp"
gh variable list
```

---

## 🐛 Debugging

**Enable debug logging for a run**

* **Why:** `ACTIONS_RUNNER_DEBUG=true` and `ACTIONS_STEP_DEBUG=true` print verbose runner and step logs — essential for diagnosing opaque failures.
* **Command:**

```bash
# Set as a secret (value: true) to enable permanently, or set for one run:
gh secret set ACTIONS_RUNNER_DEBUG --body "true"
gh secret set ACTIONS_STEP_DEBUG --body "true"
```

**Dump a context to logs for debugging**

* **Why:** Inspect the full contents of `github`, `runner`, or `needs` contexts when expressions behave unexpectedly.
* **Command (add as a step in the workflow):**

```yaml
- name: Dump github context
  run: echo '${{ toJSON(github) }}'
- name: Dump runner context
  run: echo '${{ toJSON(runner) }}'
```

**Run workflows locally with `act`**

* **Why:** Test workflow changes locally without pushing to GitHub. Saves runner minutes and speeds up iteration.
* **Command:**

```bash
# Install: https://github.com/nektos/act
brew install act

act push                              # Simulate a push event
act pull_request                      # Simulate a PR event
act workflow_dispatch --input env=staging
act push --job test                   # Run only the 'test' job
act push --secret-file .secrets       # Use a local .secrets file (KEY=value format)
act push --dry-run                    # Show what would run without executing
```

💡 **`act` caveat:** `act` uses Docker to simulate runners. Image behavior may differ from GitHub-hosted runners — use for rapid iteration, not as a replacement for real runs. Node.js actions work well; Docker actions may have layer compatibility issues.

---

## 🔑 Workflow Permissions & OIDC

**View the current `GITHUB_TOKEN` permissions for a repo**

* **Why:** Check whether the repo defaults to read or write permissions for `GITHUB_TOKEN`.
* **Command:**

```bash
gh api repos/{owner}/{repo} --jq '.default_workflow_permissions'
```

**Set default workflow permissions to read-only (org-wide security posture)**

* **Why:** Enforce least privilege by default; workflows must explicitly declare write permissions.
* **Command:**

```bash
gh api --method PUT orgs/{org}/actions/permissions/workflow \
  --field default_workflow_permissions=read \
  --field can_approve_pull_request_reviews=false
```

---

## 🏷️ Cache Management

**List caches for a repo**

* **Why:** See what's cached, how large each cache is, and when it was last accessed.
* **Command:**

```bash
gh api repos/{owner}/{repo}/actions/caches | jq '.actions_caches[] | {key, size_in_bytes, last_accessed_at}'
```

**Delete a stale or corrupted cache by key**

* **Why:** A corrupted cache can cause persistent, confusing CI failures. Delete and let it rebuild.
* **Command:**

```bash
gh api --method DELETE repos/{owner}/{repo}/actions/caches?key=Linux-go-abc123
```
