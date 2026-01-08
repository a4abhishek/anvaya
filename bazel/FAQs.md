# Bazel FAQs: Hard-Learnt Nuggets

> **The Anvaya:** *If you are fighting Bazel, you are fighting physics. Stop and fix the graph.*

---

## 🚀 Optimization & Cheats

**Q: How do I make builds faster (The Right Way)?**

* **The Strategy:** **Granularity Tuning**.
* **The Problem:** You have a `//common` target containing `small.go` (changes often) and `huge.go` (slow compile). Changing `small.go` rebuilds `huge.go`.
* **The Fix:** Split them into `//common:small` and `//common:huge`. Now, changing `small` *only* rebuilds things that depend on `small`. The `huge` compilation is cached.
* **The Metric:** Use `bazel analyze-profile` to find these "bottleneck" targets.

**Q: Why is `local_repository` dangerous (The Wrong Way)?**

* **The Cheat:** Mapping a dependency to a local folder (`path = "/Users/me/lib"`) to skip downloading or to hack on it.
* **The Result:** **Hermeticity Death.**
  * **CI Fails:** The CI runner doesn't have `/Users/me/lib`.
  * **Drift:** You are building against your uncommitted hacks, not the source of truth.
* **The Better Way:** Use `git_override` (in `MODULE.bazel`) pinned to a specific commit, or `bazel mod edit` for temporary local overrides that you *know* are temporary.

---

## 🐢 Performance & Slowness

**Q: Why use Bazel if `go build` works fine?**

* **The Reality:** Go's tooling is excellent for *just* Go.
* **The Gap:**
    1. **Polyglot:** Your app isn't just Go. It's Go + Proto + Docker + TypeScript + Terraform. `go build` can't orchestrate that DAG. Bazel can.
    2. **Hermeticity:** `go build` uses whatever `go` is in your `$PATH`. Bazel downloads a specific, hash-locked Go SDK for everyone.
    3. **Remote Caching:** Go caches locally. Bazel caches globally (S3/GCS), so your CI builds in seconds, not minutes.
    4. **Integration Tests:** Bazel can spin up a Postgres sidecar, run your Go binary against it, and tear it down, all hermetically.

**Q: Why is "Analyzing..." taking forever?**

* **The Cause:** Your build graph is too big or you have a "bottleneck" dependency (everything depends on everything).
* **The Fix:**
    1. **Check Memory:** `bazel info` -> Look at heap size. Give it more RAM: `build --host_jvm_args=-Xmx4g`.
    2. **Query the Graph:** Find the "God Node": `bazel query 'rdeps(//..., //common:util)'`. If `util` triggers a rebuild of the universe, split it up.

**Q: Why is the build slow even when nothing changed?**

* **The Cause:** Non-hermetic actions (using timestamps or external files not declared in inputs).
* **The Fix:** Run with `--explain=log.txt`. Search for "Executing action" to see *why* Bazel decided to run it.

---

## 🐛 Debugging Failures

**Q: "It works on my machine but fails in CI."**

* **The Cause:** Your machine has `/usr/include/openssl` installed. The CI sandbox does not.
* **The Fix:** You are missing a dependency in `deps`. Run `bazel build //:target --sandbox_debug` to inspect the sandbox folder and see what's actually there.

**Q: "Undeclared inclusion(s) in rule..." (C++)**

* **The Cause:** You `#include "header.h"` but didn't add the library providing it to `deps`.
* **The Fix:** Use `bazel run //:gazelle` (if supported) or manually add the library to `deps`.

---

## 📦 Dependencies & Naming

**Q: Docker BuildKit vs. rules_oci?**

* **The Conflict:** Both build container images.
* **BuildKit (Dockerfile):**
  * **Imperative:** "Run this shell command."
  * **Non-Deterministic:** `apt-get install` changes daily. Hard to reproduce old builds.
  * **Layer Caching:** Linear. Change line 1, rebuild lines 2-10.
* **rules_oci (Bazel):**
  * **Declarative:** "Assemble these files."
  * **Deterministic:** Byte-for-byte identical output. No timestamps, no `apt-get`.
  * **Layer Caching:** Independent. Changing the app binary doesn't require "re-downloading" the base OS layer.
  * **No Daemon:** Builds tarballs without needing Docker running (great for CI).

**Q: How do I know the dependency name (e.g., `@org_uber_go_zap`)?**

* **The Problem:** You know the Go import (`go.uber.org/zap`), but not the Bazel label.
* **The Solution (Go):**
  * **Don't guess.** Let Gazelle do it.
  * **Technique:** Run `bazel run //:gazelle`. Gazelle maps imports to labels automatically.
  * **Manual Lookup:** If you *must* know, `rules_go` usually replaces `/` and `.` with `_`. (e.g., `github.com/pkg/errors` -> `@com_github_pkg_errors//:errors`).
* **The Solution (Bzlmod):**
  * Search the [Bazel Central Registry (BCR)](https://registry.bazel.build/). The name listed there (e.g., `rules_go`) is the name you use in `bazel_dep`.
* **The Trick:** Run `bazel mod graph` to see the canonical names of all repos in your graph.

**Q: WORKSPACE vs Bzlmod?**

* **The Answer:** Bzlmod is the future. WORKSPACE is the past.
* **The Detail:** See **[Workspace vs Bzlmod](Workspace-vs-Bzlmod.md)** for the side-by-side comparison.

**Q: How do I use a private Git repo?**

* **The Fix:** Use `git_override` in `MODULE.bazel`.

    ```starlark
    git_override(
        module_name = "my_internal_lib",
        remote = "ssh://git@github.com/company/lib.git",
        commit = "a1b2c3d...",
    )
    ```

---

## 🛠️ Daily Life

**Q: Will Bazelisk overwrite my system Bazel?**

* **The Answer:** **No.**
* **The Detail:** Bazelisk is a "launcher." It downloads the version requested in `.bazelversion` to a private cache (e.g., `~/.bazelisk/bin/`) and executes it from there. Your system-wide Bazel (e.g., in `/usr/bin/bazel`) remains untouched.
* **✨ BEST PRACTICE:** Alias `bazel=bazelisk` in your `.bashrc` or `.zshrc`. This ensures you *always* use the project-defined version without ever worrying about what's installed globally.

**Q: Is "common" a special name in Bazel?**

* **The Answer:** It depends on the file.
* **In `.bazelrc`:** **YES.** It is a reserved prefix. `common --color=yes` applies that flag to *every* command (`build`, `test`, `query`, etc.).
* **In `BUILD.bazel`:** **NO.** It is just a name. You can name a library `:common`, `:util`, or `:banana`. Bazel doesn't care.

**Q: Does Gazelle only work for Go?**

* **The Answer:** No. Gazelle is a **pluggable framework**.
* **The Detail:** While it was born for Go, it now supports many languages via **extensions**:
  * **Go:** Built-in (the gold standard).
  * **Python:** Supported via the `rules_python` Gazelle extension.
  * **Java:** Supported via `rules_java` extensions (though many still use `bazel-deps`).
  * **Others:** Protobuf, JavaScript/TypeScript, and even CSS extensions exist.
* **The Anvaya:** If an extension exists for your language, **use it**. Manual `BUILD` maintenance is a path to architectural drift.

**Q: How do I format BUILD files?**

* **The Command:** `bazel run //:buildifier` (requires `buildifier` setup).

**Q: How do I run a specific test case?**

* **The Command:** `bazel test //pkg:test --test_filter="TestName"`

**Q: How do I clean the cache?**

* **The Command:** `bazel clean --expunge`.
* **The Warning:** This deletes *everything* (downloaded repos, toolchains). It will take 10 minutes to re-fetch. Use only as a last resort.
