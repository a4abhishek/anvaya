# Bazel: The Magic Spells

> **The Anvaya:** *Memorize the patterns, not the flags. These are the incantations for daily survival.*

## 🏗️ Workspace & Maintenance

**Refresh Dependencies**

* **Why:** Update `MODULE.bazel.lock` and fetch new repos.
* **Command:** `bazel mod sync_all_repos`

**Auto-generate BUILD files (Gazelle)**

* **Why:** Fix missing dependencies or new files.
* **Command:** `bazel run //:gazelle`

**Update Gazelle Repos**

* **Why:** Add new external Go modules to `deps.bzl` (or equivalent).
* **Command:** `bazel run //:gazelle-update-repos`

**Clean Everything**

* **Why:** Nuke the cache (last resort for "ghosts").
* **Command:** `bazel clean --expunge`

---

## 🔍 Graph Querying (Architecture)

**Find Reverse Dependencies**

* **Why:** "What breaks if I change this target?"
* **Command:** `bazel query 'rdeps(//..., //backend/logger)'`

**Find Path Between Targets**

* **Why:** "Why does Service A depend on Library B?"
* **Command:** `bazel query 'somepath(//service/a, //library/b)'`

**Show Build Graph**

* **Why:** Visualize the mess (requires Graphviz).
* **Command:** `bazel query --output=graph //... | dot -Tpng > graph.png`

**Debug Cross-Compilation**

* **Why:** Check flags for a specific platform.
* **Command:** `bazel cquery //cmd/server --platforms=@io_bazel_rules_go//go/toolchain:linux_amd64`

---

## 🐞 Testing & Debugging

**Run All Tests**

* **Why:** The standard CI check.
* **Command:** `bazel test //...`

**Detect Flaky Tests**

* **Why:** Run 100 times to prove stability.
* **Command:** `bazel test //pkg/api:test --runs_per_test=100`

**Stream Output (No Buffering)**

* **Why:** See logs in real-time (for hanging tests).
* **Command:** `bazel test //pkg/api:test --test_output=streamed`

**Inspect Sandbox**

* **Why:** Debug "Works locally, fails in CI".
* **Command:** `bazel build //pkg/api:target --sandbox_debug`

---

## 📦 Packaging & Run

**Build OCI Image**

* **Why:** Create the tarball (don't load to Docker yet).
* **Command:** `bazel build //cmd/server:image`

**Run Locally (Fast)**

* **Why:** Execute the binary directly (no container).
* **Command:** `bazel run //cmd/server`

**Pass Arguments to Binary**

* **Why:** Send flags to the running app.
* **Command:** `bazel run //cmd/server -- --port=8080`

---

## 🐍 Python Specifics

**Update Python Lockfile**

* **Why:** Re-resolve `requirements.txt` into a frozen lockfile.
* **Command:** `bazel run //:requirements.update`

**Run Python REPL**

* **Why:** Get an interactive shell with all `deps` available.
* **Command:** `bazel run //path/to:my_python_binary -- --repl`

**Test Specific Python File**

* **Why:** Run a single test instead of the whole suite.
* **Command:** `bazel test //path/to:test --test_arg=-k --test_arg=test_name`
