# GitHub Actions: Concepts

> **The Anvaya:** *Every GitHub Actions feature — runners, secrets, OIDC, contexts — is a named component with a specific scope and lifecycle; knowing the boundaries removes the mystery.*

---

## **<a id="workflow"></a>Workflow**

> A YAML file in `.github/workflows/` that defines an automated process triggered by events.

- A repo can have unlimited workflows. Each `.yml` file is an independent workflow.
- Workflows run in isolation from each other (no shared state by default).
- Visible under the "Actions" tab. Each run is logged, queryable, and re-runnable.

> 💡**PRO TIP:** State CAN be shared across runs via:
>
> - **Artifacts** (`actions/upload-artifact` / `download-artifact` — files scoped to a run, downloadable by later runs in the same repo),
> - **Cache** (`actions/cache` — keyed blobs persisting across runs),
> - **Repository Variables** (`vars` context — non-secret config), or
> - **external storage** (S3, a DB, etc. — the only option that works cross-repo).

---

## **<a id="event"></a>Event**

> The thing that causes a workflow to start — a push, a pull request, a schedule, a manual click, or a webhook.

- `push`, `pull_request`, `release` — Git events.
- `schedule` (cron), `workflow_dispatch` (manual) — time or human-triggered.
- `workflow_call` — triggered by another workflow. Makes a workflow **reusable**.
- `repository_dispatch` — triggered by an external HTTP POST to the GitHub API.

---

## **<a id="job"></a>Job**

> A collection of Steps that runs on a single Runner. Jobs in the same workflow run in parallel by default.

- Each Job gets a **fresh runner instance**. No filesystem state persists between jobs.
- `needs:` creates a directed acyclic graph (DAG) of jobs. A job waits for its `needs` to complete successfully.
- A job can be skipped with `if:` conditions — it counts as "success" for dependent jobs.
- `if: always()` runs a job truly **regardless** of prior state — including cancellation. `if: failure()` only triggers when a prior job actually **failed** — it does **not** run on cancellation. Use `always()` for cleanup; use `failure()` for failure-specific alerting or notifications.

---

## **<a id="step"></a>Step**

> A single unit of work inside a Job — either a shell command (`run:`) or an Action (`uses:`).

- Steps in a Job run **sequentially** on the same runner, sharing the same filesystem.
- Each step can be individually named, conditioned (`if:`), and its output captured (`id:` + `$GITHUB_OUTPUT`).
- A failed step fails the job. Use `continue-on-error: true` to override this for non-critical steps.

```yaml
steps:
  - name: Compute version
    id: version
    run: echo "value=1.2.3" >> $GITHUB_OUTPUT

  - name: Use version
    run: echo "Version is ${{ steps.version.outputs.value }}"
```

---

## **<a id="action"></a>Action**

> A reusable, packaged automation unit invoked with `uses:`. Can be a Docker container, a JavaScript program, or a composite of shell steps.

- **Published Action:** Hosted in a GitHub repo, referenced as `owner/repo@version`. Examples: `actions/checkout@v4`, `aws-actions/configure-aws-credentials@v4`.
- **Composite Action:** A local `action.yml` using `runs: using: composite` — a set of Steps bundled for reuse within a repo.
- **Docker Action:** Runs a Docker container as a step. Slower startup, but fully isolated environment.
- Action versions can be a tag (`@v4`), branch (`@main`), or SHA (`@11bd71901bbe...`). **SHA is the only immutable reference** — use it for security.

---

## **<a id="runner"></a>Runner**

> The compute that executes a Job. GitHub-hosted or self-hosted.

- **GitHub-hosted:** Ephemeral VMs spun up fresh per Job. Auto-provisioned, maintained by GitHub. Available for `ubuntu-*`, `windows-*`, `macos-*`.
- **Self-hosted:** Your own machine/container registered with GitHub. Full control of resources and network — required for private network access. Risk: not ephemeral by default, so prior-run state can persist.
- Larger GitHub-hosted runners (4+ cores, more RAM) are available but cost more.
- `runs-on: [self-hosted, linux, x64]` uses label-based runner selection for self-hosted.

**Making Self-Hosted Runners Ephemeral**

- **`--ephemeral` flag:** Pass during registration (`./config.sh --ephemeral`). The runner picks up exactly one job then deregisters itself. Simplest option.
- **JIT (Just-In-Time) runners:** GitHub API generates a single-use registration token per job. Used by all auto-scaling solutions — the runner bootstraps, runs one job, and the instance terminates.
- **Container/VM isolation:** Wrap the runner in a Docker container or VM booted from a clean image. Destroy the instance after the job completes. Tools: `myoung34/docker-github-actions-runner`.

⚠️ **ANTI-PATTERN:** Without `--ephemeral` or container isolation, a compromised job can plant files, env vars, or background processes that persist for all future jobs on that runner.

**Self-Hosted Runner Security**

- **Never use self-hosted with public repos** unless every PR author is trusted — forks can run arbitrary code on your machine.
- **Network isolation:** Run in a private subnet. Restrict egress to GitHub's IP ranges and your deployment targets only.
- **Least-privilege IAM:** Attach an IAM instance profile with only what the job needs. No long-lived AWS keys — use OIDC or the instance role.
- **Dedicated OS user:** Run the runner as a non-root service account. No sudo, no access to other workloads on the host.
- **No secrets on disk:** Don't bake credentials into the runner AMI/image. Fetch at runtime via SSM Parameter Store or Secrets Manager.
- **Audit logging:** Enable CloudTrail + GitHub's audit log streaming. Know what ran on your runner and when.

**AWS-Based GitHub Runners**

Three common patterns:

| Pattern | How | Cost model | Best for |
| :--- | :--- | :--- | :--- |
| **EC2 self-hosted (static)** | Register EC2 instance with `--ephemeral` | Always-on EC2 | Steady, predictable job volume |
| **Auto-scaling on EC2** | GitHub webhook → Lambda → Spot EC2 via [`philips-labs/terraform-aws-github-runner`](https://github.com/philips-labs/terraform-aws-github-runner) | Per-job Spot pricing | Variable load, cost-sensitive |
| **AWS CodeBuild Runner** | `runs-on: codebuild-<project>-${{ github.run_id }}` (native GitHub integration, GA 2024) | Per-build-minute | Serverless — zero runner management |

- **Auto-scaling (philips-labs):** Webhook fires → Lambda registers a JIT token → Spot EC2 boots → runs the job → terminates. Fully ephemeral. Most popular OSS auto-scaling approach.
- **CodeBuild:** Native GitHub integration. Define a CodeBuild project (custom image, ARM, GPU supported), reference it in `runs-on`. No runner agent to manage.

✨ **BEST PRACTICE:** Pair CodeBuild or auto-scaling EC2 with OIDC. Zero stored AWS credentials, zero always-on runner cost, fully ephemeral compute.

---

## **<a id="context"></a>Context**

> A named object containing runtime metadata about the workflow run — available via the `${{ }}` expression syntax.

| Context | Contains |
| :--- | :--- |
| `github` | Event name, repo, ref, SHA, actor, run ID |
| `env` | Environment variables set in the workflow |
| `vars` | Repository/org-level variables (non-secret) |
| `secrets` | Encrypted secrets — value never printed in logs |
| `job` | Current job status, container info |
| `steps` | Outputs and outcome of previously run steps |
| `runner` | OS, arch, temp dir of the runner |
| `needs` | Outputs + result of jobs listed in `needs:` |
| `inputs` | Inputs passed to `workflow_call` or `workflow_dispatch` |

- Contexts are read-only. You cannot set `github.sha` inside a workflow.
- Use the `toJSON()` function to debug a context: `run: echo '${{ toJSON(github) }}'`

---

## **<a id="expressions"></a>Expressions**

> The `${{ }}` syntax used in YAML values to compute values at runtime using contexts, functions, and operators.

- Used in: `if:`, env values, `with:` inputs, `run:` (for non-shell injection — prefer `env:` instead).
- Status functions: `success()`, `failure()`, `cancelled()`, `always()` — evaluate the state of prior steps/jobs.
- Common functions: `hashFiles('**/go.sum')`, `toJSON(context)`, `fromJSON(string)`, `contains(str, sub)`, `startsWith(str, prefix)`.
- Ternary pattern: `${{ condition && 'value-if-true' || 'value-if-false' }}`

---

## **<a id="github-token"></a>GITHUB_TOKEN**

> An auto-generated, short-lived token scoped to the current workflow run. No setup required.

- Can be used to push commits, create PRs, publish packages, comment on issues — all within the same repo.
- Automatically created at the start of each run. Expires when the run ends.
- Scopes are controlled by `permissions:` in the workflow YAML (repo settings can further restrict it).
- **Does NOT work across repos.** Use a PAT or App installation token for cross-repo operations.
- Default permissions changed in ~2022: many repos now default to **read** for most scopes. Check your org settings.

```yaml
- name: Create release
  uses: softprops/action-gh-release@v2
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}    # Auto-available — no setup
```

---

## **<a id="oidc"></a>OIDC (OpenID Connect)**

> A protocol where GitHub Acts as an identity provider, issuing short-lived JWTs that cloud providers (AWS, GCP, Azure) accept instead of static credentials.

- **The flow:** GitHub runner requests a JWT → sends it to AWS → AWS validates the JWT against GitHub's OIDC endpoint → issues a short-lived role credential.
- **Why it matters:** No secrets stored. Credentials expire in minutes. No rotation required.
- **Prerequisites:** Cloud IAM must be pre-configured with a trust policy for your GitHub repo/branch/environment.
- Requires `permissions: id-token: write` on the job.

```
Trust policy condition (AWS):
  github.com/YOUR_ORG/YOUR_REPO:ref:refs/heads/main
  → Only grants credentials to main branch runs of that specific repo.
```

---

## **<a id="environment"></a>Environment**

> A named deployment target (e.g., `staging`, `prod`) with its own secrets, variables, and protection rules.

- Environments live at repo or org level. Created under **Settings → Environments**.
- Protection rules: **required reviewers** (up to 6 people or teams), **wait timer** (up to 30 days), **branch restrictions** (only specific branches can deploy to it).
- Secrets and variables scoped to an environment are **only accessible to jobs that declare `environment: <name>`**.
- Deployments to a named environment are tracked in the GitHub "Deployments" tab — creates an audit trail.

---

## **<a id="concurrency"></a>Concurrency**

> A mechanism to prevent redundant parallel runs and eliminate deployment races.

- `concurrency.group:` defines a named mutex. Only one workflow with the same group key runs at a time.
- `cancel-in-progress: true` cancels the running workflow when a new one queues for the same group.
- `cancel-in-progress: false` queues the new run, waiting for the current to finish.

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true   # Best for CI/build; use false for deploy to avoid torn state
```

- Common group patterns: `deploy-${{ github.ref }}` (one deploy per branch), `pr-${{ github.event.pull_request.number }}` (one CI run per PR).

---

## **<a id="reusable-workflow"></a>Reusable Workflow**

> A workflow file triggered by `on: workflow_call` — callable from other workflows using `uses: ./.github/workflows/name.yml`.

- Runs its own Jobs on its own Runners — full job isolation.
- Inputs typed as `string`, `boolean`, `number`. Secrets must be explicitly declared and passed.
- Output values can be declared on the reusable workflow's jobs and accessed by the caller via `needs`.
- Called with `uses:` in a Job (not a Step). Cannot use `runs-on:` in the caller for the called job.
- `secrets: inherit` passes all caller secrets to the reusable workflow — convenient but reduces explicitness; prefer explicit secret passing in production.

```yaml
# .github/workflows/deploy-template.yml  (the reusable workflow)
on:
  workflow_call:
    inputs:
      environment:
        type: string
        required: true
    secrets:
      deploy-key:
        required: true
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - run: ./deploy.sh
        env:
          DEPLOY_KEY: ${{ secrets.deploy-key }}
```

```yaml
# .github/workflows/release.yml  (the caller)
jobs:
  deploy-prod:
    uses: ./.github/workflows/deploy-template.yml
    with:
      environment: production
    secrets:
      deploy-key: ${{ secrets.DEPLOY_KEY }}
```

---

## **<a id="composite-action"></a>Composite Action**

> A local `action.yml` that bundles multiple steps into a reusable unit invoked with `uses:` inside a job.

- Lives in `.github/actions/<name>/action.yml` (local) or a separate repo (published).
- Runs within the **calling job's runner** — shares filesystem, env vars.
- Inputs declared under `inputs:`. No first-class secret inputs — pass secrets via env on the calling step.
- No `jobs:` section. Just `runs: using: composite` and a `steps:` list.
- Lighter than a full reusable workflow (no extra runner spin-up). Preferred for step-level reuse.

```yaml
# .github/actions/setup-go-and-cache/action.yml
name: Setup Go with Cache
description: Install Go and restore module cache
inputs:
  go-version:
    required: true
runs:
  using: composite
  steps:
    - uses: actions/setup-go@v5
      with:
        go-version: ${{ inputs.go-version }}
    - uses: actions/cache@v4
      with:
        path: ~/go/pkg/mod
        key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
        restore-keys: ${{ runner.os }}-go-
```

```yaml
# Usage in any workflow job — no extra runner spin-up
steps:
  - uses: actions/checkout@v4
  - uses: ./.github/actions/setup-go-and-cache
    with:
      go-version: '1.22'
  - run: go test ./...
```
