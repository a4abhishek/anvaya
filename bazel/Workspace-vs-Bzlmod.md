# Bzlmod vs. WORKSPACE: The Migration

> **The Anvaya:** *WORKSPACE was a list of downloads. Bzlmod is a dependency graph.*

---

## 🏗️ The Fundamental Shift

**Legacy (`WORKSPACE`)**

* **Philosophy:** "Download this URL and unzip it here."
* **Resolution:** Flat. If Repo A needs `guava-20` and Repo B needs `guava-30`, you (the human) must manually pick one and hope it works for both.
* **Pain Point:** "Diamond Dependency Hell."

**Modern (`MODULE.bazel`)**

* **Philosophy:** "I need this module, version X."
* **Resolution:** Transitive & Semantic. Bazel uses Minimal Version Selection (MVS) to mathematically pick a single compatible version for the entire graph.
* **Gain:** Automatic conflict resolution.

---

## ⚔️ Side-by-Side Comparison

### 1. Declaring a Dependency

**Legacy (WORKSPACE)**

```starlark
# WORKSPACE
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
    name = "rules_go",
    sha256 = "692fb...",
    urls = ["https://github.com/bazelbuild/rules_go/releases/download/v0.41.0/rules_go-v0.41.0.zip"],
)

load("@rules_go//go:deps.bzl", "go_rules_dependencies", "go_register_toolchains")
go_rules_dependencies()
go_register_toolchains(version = "1.20.7")
```

**Modern (Bzlmod)**

```starlark
# MODULE.bazel
bazel_dep(name = "rules_go", version = "0.50.1")

go_sdk = use_extension("@rules_go//go:extensions.bzl", "go_sdk")
go_sdk.download(version = "1.22.0")
use_repo(go_sdk, "go_default_sdk")
```

### 2. Transitive Dependencies

**Legacy (WORKSPACE)**

* **You:** "I need `rules_docker`."
* **Bazel:** "Okay, but `rules_docker` needs `rules_python` and `rules_go`. Please copy-paste those 50 lines into your WORKSPACE too."
* **Result:** A 2000-line `WORKSPACE` file full of junk you don't recognize.

**Modern (Bzlmod)**

* **You:** `bazel_dep(name = "rules_docker", version = "0.25.0")`
* **Bazel:** "I see it needs python and go. I'll fetch them automatically."
* **Result:** A clean, readable 10-line file.

---

## 🛡️ The Registry (BCR)

**Legacy:**
You trusted random URLs on GitHub. If the author deleted the release, your build broke.

**Modern:**
Bzlmod uses the **Bazel Central Registry (BCR)**.

* **Immutable:** Versions are indexed and hash-locked.
* **Discoverable:** You can search for modules (like npm or maven).
* **Secure:** No more "random zip file from the internet."

---

## 🔄 Migration Strategy

1. **Don't Rewrite Yet:** You can use both! Bzlmod is enabled by default in Bazel 7+, but it respects `WORKSPACE` for things it can't find.
2. **Enable Bzlmod:** `common --enable_bzlmod` (Default in v7).
3. **Migrate One by One:** Move `rules_go`, then `rules_python`, checking that builds pass at each step.
4. **Delete WORKSPACE:** The ultimate goal.
