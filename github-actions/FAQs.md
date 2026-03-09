# GitHub Actions: FAQs

> **The Anvaya:** *Most GitHub Actions confusion comes from misunderstanding scopes — of secrets, tokens, triggers, and contexts.*

---

## 🏛️ Triggers & Events

**Q: My workflow triggered when I didn't expect it. Why?**

`pull_request` defaults to `types: [opened, synchronize, reopened]`. Any commit pushed to an open PR triggers it. If you're seeing unexpected runs, also check that you have no branch filter — `on: push` without `branches:` fires on every branch including `wip/` and `dependabot/*`.

```yaml
# Add filters to prevent surprise triggers:
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
    paths-ignore: ["**.md"]
```

**Q: How do I skip CI for a specific commit?**

Add `[skip ci]`, `[ci skip]`, `[no ci]`, or `[skip actions]` anywhere in the commit message. GitHub natively skips all workflows triggered by `push` and `pull_request` for that commit.

```bash
git commit -m "fix typo in README [skip ci]"
```

**Q: My `workflow_dispatch` doesn't appear in the UI. Why?**

The workflow file containing `on: workflow_dispatch` must exist on the **default branch** (usually `main`). GitHub only shows the manual trigger button for workflows on the default branch. A workflow on a feature branch won't show the button even if the `on: workflow_dispatch` trigger is defined.

---

## 🏛️ Secrets & Credentials

**Q: `${{ secrets.MY_SECRET }}` shows `***` in the log but my step still fails. What's wrong?**

The secret exists and is being masked, but the step is failing for another reason. Common ones:

1. The secret value has **leading/trailing whitespace** — copy-pasted with an extra newline.
2. It's a multi-line secret (PEM key) — use `base64`-encoded secrets and decode at runtime.
3. The secret is set at **environment** level but the job doesn't declare `environment:`.

Check by printing the secret's length (not value) to confirm it's non-empty:

```bash
echo "Secret length: ${#SECRET}" | sed 's/[0-9]\+/[REDACTED]/g'
```

**Q: How do I pass a secret to a reusable workflow?**

Secrets must be explicitly declared in the reusable workflow and passed by the caller. They are **not** automatically inherited.

```yaml
# Reusable workflow:
on:
  workflow_call:
    secrets:
      deploy_key:
        required: true

# Caller:
jobs:
  call-deploy:
    uses: ./.github/workflows/_deploy.yml
    secrets:
      deploy_key: ${{ secrets.PROD_DEPLOY_KEY }}
    # OR use `secrets: inherit` to pass all caller secrets
```

**Q: `GITHUB_TOKEN` fails when I try to push to a protected branch. Why?**

Protected branches with **"Require pull request reviews"** or **"Restrict who can push"** block `GITHUB_TOKEN` pushes by default. Options:

1. Add a **GitHub App** or **Deploy Key** with write access and use its token.
2. In the branch protection rule, enable **"Allow specified actors to bypass required pull requests"** and add the `github-actions[bot]` app.
3. Restructure to avoid direct pushes from CI — use PRs instead.

**Q: My OIDC role assumption fails with `Not authorized to assume role`. Why?**

The AWS IAM trust policy condition doesn't match your workflow's context. Check:

1. **Branch condition:** Policy says `ref:refs/heads/main` but you're running on a PR branch.
2. **Environment condition:** Policy requires `environment:prod` but the job has no `environment:` key.
3. **Repository condition:** Typo in org/repo name in the trust policy.

Print the OIDC JWT claims to debug: `run: curl -H "Authorization: Bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=sts.amazonaws.com" | jq '.value' | cut -d. -f2 | base64 -d | jq`

---

## 🏛️ Jobs, Steps & Outputs

**Q: Job B can't read job A's output. Why?**

Two things must be true: (1) the source job must declare `outputs:` at the job level, AND (2) the source step must write to `$GITHUB_OUTPUT`. Missing either one breaks it. Also confirm job B has `needs: [job_a]`.

```yaml
jobs:
  build:
    outputs:
      tag: ${{ steps.compute.outputs.tag }}   # (1) Job-level declaration
    steps:
      - id: compute
        run: echo "tag=1.0.0" >> $GITHUB_OUTPUT   # (2) Step writes to GITHUB_OUTPUT

  deploy:
    needs: [build]
    steps:
      - run: echo ${{ needs.build.outputs.tag }}  # (3) Consumer
```

**Q: How do I run a cleanup job even when another job fails?**

Use `if: always()` on the cleanup job. Without it, a cleanup job is skipped when its `needs` fails.

```yaml
  cleanup:
    needs: [build, test, deploy]
    if: always()    # Runs regardless of build/test/deploy outcome
    steps:
      - run: ./cleanup.sh
```

**Q: My matrix job fails but I want to see all results before the workflow is marked failed. How?**

Set `fail-fast: false` on the strategy:

```yaml
strategy:
  fail-fast: false   # Default is true — cancels remaining matrix jobs on first failure
  matrix:
    os: [ubuntu-24.04, macos-14]
```

---

## 🏛️ Caching

**Q: My cache never hits even though nothing changed. Why?**

Three common reasons:

1. **Key mismatch:** Cache keys are case-sensitive and exact. `${{ runner.os }}` = `Linux`, not `linux`.
2. **Cache stored on wrong branch:** Caches are branch-local, with fallback to the default branch. A cache stored on `feature/x` won't be found from `main` (but `main` cache is accessible from any branch).
3. **`hashFiles` glob mismatch:** `hashFiles('**/go.sum')` returns empty string if the path matches nothing — meaning the key is constant and may conflict.

Debug: `actions/cache@v4` logs "Cache hit" or "Cache miss" with the exact key it searched for.

**Q: My cache is getting very large. How do I control it?**

- GitHub per-repo cache limit: **10 GB**. Oldest caches are evicted first.
- Each unique key creates a new cache entry — use `restore-keys:` fallback instead of embedding commit SHA in keys.
- Use `path:` to cache only what matters (e.g., `~/.cache/go-build` for Go, not the whole home directory).

---

## 🏛️ Security

**Q: What exactly is the `pull_request_target` security trap?**

`pull_request_target` runs in the context of the **base repo** (not the fork), which means it has access to secrets and write `GITHUB_TOKEN`. If you check out and execute code from the PR's head SHA in this context, an attacker submits a PR that modifies workflow files or scripts, and your CI executes their code with your production secrets.

```yaml
# ❌ DANGEROUS: attacker code runs with your secrets
on: pull_request_target
steps:
  - uses: actions/checkout@v4
    with:
      ref: ${{ github.event.pull_request.head.sha }}
  - run: ./scripts/build.sh   # Attacker controls this file
```

Only use `pull_request_target` for things that genuinely need base repo secrets (like posting comments) and **never** check out or execute PR code in those steps.

**Q: An action I use got compromised via a tag update. How do I prevent that?**

Pin all third-party actions by their full commit SHA, not by tag. Tags are mutable — `@v4` can be moved to point at malicious code. A commit SHA is immutable.

```yaml
# ❌ Vulnerable: tag can be moved
- uses: actions/checkout@v4

# ✅ Immutable: this exact commit is pinned forever
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
```

Use `github.com/renovateapp/renovate` or `github.com/dependabot` to automatically propose SHA updates with changelogs.
