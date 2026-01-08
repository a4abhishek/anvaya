# Bazel Optimization: Code Examples

> **The Anvaya:** *Code tells the truth. See the difference between a cache hit and a rebuild.*

## ✅ The Right Way: Granularity Tuning

**Scenario:** You have a utility package. `strings.go` changes daily. `math.go` is complex and changes yearly.

### 🔴 Before (Monolith)

* **Impact:** Changing `strings.go` forces a recompilation of `math.go`.

```starlark
# common/BUILD.bazel
go_library(
    name = "common",
    srcs = [
        "strings.go", # Light, changes often
        "math.go",    # Heavy, changes rarely
    ],
    importpath = "github.com/org/repo/common",
)
```

### 🟢 After (Fine-Grained)

* **Impact:** Changing `strings.go` creates a new cache key for `:strings`, but `:math` remains a valid **Cache Hit**.

```starlark
# common/BUILD.bazel

go_library(
    name = "strings",
    srcs = ["strings.go"],
    importpath = "github.com/org/repo/common/strings",
)

go_library(
    name = "math",
    srcs = ["math.go"],
    importpath = "github.com/org/repo/common/math",
    # Note: If math needs strings, declare it.
    # If they are independent, they build in parallel!
    deps = [":strings"],
)
```

---

## ❌ The Wrong Way: Breaking Hermeticity

**Scenario:** You want to use your local fork of `rules_go`.

### 🔴 The Anti-Pattern (Local Override)

* **Why it's bad:** CI will fail immediately because `/Users/abhishek` does not exist on the runner.

```starlark
# MODULE.bazel
bazel_dep(name = "rules_go", version = "0.50.1")

# ⚠️ DANGER: Works on my machine, fails everywhere else
local_path_override(
    module_name = "rules_go",
    path = "/Users/abhishek/dev/rules_go",
)
```

### 🟢 The Safe Fix (Git Override)

* **Why it's good:** It points to a specific commit that **everyone** (including CI) can reach.

```starlark
# MODULE.bazel
bazel_dep(name = "rules_go", version = "0.50.1")

# ✅ SAFE: Pinned to a reachable commit hash
git_override(
    module_name = "rules_go",
    remote = "https://github.com/my-fork/rules_go.git",
    commit = "a1b2c3d4e5f6...",
)
```
