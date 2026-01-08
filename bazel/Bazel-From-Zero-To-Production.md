# Learning Bazel: From Zero to Production

> **The Anvaya:** *Bazel is not a build tool; it is a queryable database of your build graph where "build" is just a side-effect of correctness.*

## 🪝 The Hook

You changed a single line of code in a 2-million-line monorepo, and CI took 45 minutes because `make` couldn't prove what *didn't* need rebuilding.

## **Phase 1: The Hermetic Seal - Defining the Workspace**

**Goal:** Establish the universe. Unlike `npm` or `pip` which rely on the host OS, Bazel defines its own reality.

### **Step 1: The Modern Standard (Bzlmod)**

Forget `WORKSPACE`. Since Bazel 7+, **Bzlmod** (modern dependency management) [[?](Concepts.md#bzlmod)] is the strict, transitive-dependency-aware standard.

```starlark
# MODULE.bazel
# The root definition of your "universe"
module(
    name = "anvaya_monorepo",
    version = "1.0.0",
)

# Core dependencies (Bazel Central Registry)
bazel_dep(name = "rules_go", version = "0.50.1")
bazel_dep(name = "gazelle", version = "0.38.0")
```

**Establish the Version Lock:**

```bash
# .bazelversion
7.4.1
```

✨ **BEST PRACTICE: Enforce the Version**
**Bazelisk** (a version-aware launcher) [[?](Concepts.md#bazelisk)] reads `.bazelversion` and downloads the exact binary. This guarantees that Developer A (Mac) and CI (Linux) use the *exact* same logic.

⚠️ **ANTI-PATTERN: Using `WORKSPACE` files**

* **Don't:** Mix `WORKSPACE` and `MODULE.bazel`.
* **Why:** `WORKSPACE` (legacy) has flat dependency resolution (diamond dependency hell). Bzlmod solves this.

**What You Learned:**

* ✅ `MODULE.bazel` is the entry point.
* ✅ The host OS is "hostile"; we declare everything we need.
* ✅ Bazelisk + `.bazelversion` = reproducible engine.

## **Phase 2: Granularity & The Build Graph**

**Goal:** Teach Bazel how to view your code. Not as files, but as a Directed Acyclic Graph (DAG) of **Targets** (items in the build graph) [[?](Concepts.md#target)].

### **Step 2: Defining Targets**

In Bazel, a directory is just a directory. It becomes a **Package** (a directory with a `BUILD` file) [[?](Concepts.md#package)] only when you add a `BUILD.bazel` file.

```starlark
# backend/logger/BUILD.bazel

load("@rules_go//go:def.bzl", "go_library")

# 🏛️ CONVENTION/PATTERN: One-to-one mapping
# The rule name "logger" matches the directory name.
go_library(
    name = "logger",
    srcs = ["logger.go"],
    importpath = "github.com/a4abhishek/anvaya/backend/logger",
    visibility = ["//backend:__subpackages__"], # 🔒 Restricted access
    deps = [
        "@org_uber_go_zap//:zap", # External dependency
    ],
)
```

💡 **TIP: Visibility is Security**
By default, targets should be `private`. Only expose what is necessary.

* `//visibility:public`: Open to the world (Anti-pattern for internal libs).
* `//backend:__subpackages__`: Open to your own backend microservices.

**The "Magic" of Sandboxing:**
When Bazel runs this compile, it creates a temporary **Sandbox** (isolated execution environment) [[?](Concepts.md#sandbox)] containing *only* `logger.go` and the declared `deps`. If `logger.go` imports `fmt` but you forgot to declare it (in some languages), the build fails. **Hidden dependencies are impossible.**

💡 **TIP: The "It Works Locally" Trap**
If a build fails in CI but works locally (or vice versa), it's a hermeticity leak. Run with `--sandbox_debug` to prevent Bazel from deleting the sandbox temp directory on failure, allowing you to inspect exactly what files were (or weren't) present.

## **Phase 3: Automated BUILD Management (Gazelle)**

**Goal:** Humans shouldn't write `BUILD` files manually. It's toil.

### **Step 3: Generating the Graph**

For languages like Go, Python, and Java, use generators.

```starlark
# MODULE.bazel
# Register the Go toolchain (hermetic compiler)
use_repo(
    bazel_dep(name = "rules_go", version = "0.50.1"),
    "go_sdk",
)
register_toolchains("@go_sdk//:all")
```

**The Snippet:**

```bash
# Initialize Gazelle
bazel run //:gazelle
```

🏛️ **CONVENTION/PATTERN: The "Fix" Loop**
Never manually edit `srcs` lists.

1. Write code (`main.go`).
2. Run `bazel run //:gazelle`.
3. Gazelle updates `BUILD.bazel`.
4. Run `bazel build //...`.

**What You Learned:**

* ✅ `BUILD` files describe the graph.
* ✅ `visibility` controls architectural boundaries.
* ✅ **Toolchains** (hermetic compilers) [[?](Concepts.md#toolchain)] are fetched, not assumed from the OS.

## **Phase 4: Hermetic Testing & Caching**

**Goal:** Run tests that never flake and never run twice if code hasn't changed.

### **Step 4: The Cache Hit**

```bash
$ bazel test //backend/...
INFO: Found 15 targets...
INFO: Elapsed time: 0.123s, Critical Path: 0.01s
//backend/logger:logger_test    (cached) PASSED
```

**Why it's cached:**
Bazel hashes:

1. The input files (`srcs`).
2. The configuration (flags).
3. The toolchain (compiler version).
4. The command line.

If the hash matches the cache, **execution is skipped.**

💡 **TIP: Flaky Test Detection**

```bash
# Run the test 100 times to prove stability
bazel test //backend/logger:logger_test --runs_per_test=100
```

⚠️ **ANTI-PATTERN: `t.Now()`**
Tests that rely on system time or external networks break hermeticity.

* **The Fix:** Inject time as a dependency. Block network access in the sandbox (`tags = ["block-network"]`).

## **Phase 5: Querying the Graph (SRE Superpower)**

**Goal:** Answer complex architectural questions instantly.

### **Step 5: The Query Language**

Since the build is a graph, we can query it.

**Scenario:** "I am changing the `logger` library. Which services will break?"

```bash
# "Find the reverse dependencies (rdeps) of logger
# in the universe (//...) of type go_binary"
bazel query 'kind("go_binary", rdeps(//..., //backend/logger:logger))'
```

**Scenario:** "Show me the dependency path between Service A and Library B."

```bash
bazel query 'somepath(//backend/service_a, //lib/dangerous_lib)'
```

✨ **BEST PRACTICE: `cquery` (Configured Query)**
`bazel query` ignores build flags (OS/Arch). `bazel cquery` respects them. Use `cquery` when debugging cross-compilation issues (e.g., "Why is this building for Linux on my Mac?").

## **Phase 6: Production Containers (Rules OCI)**

**Goal:** Create "Distroless" images. Byte-for-byte reproducible.

### **Step 6: No Dockerfiles**

Dockerfiles are imperative (run this, then that). They are non-deterministic (apt-get update changes daily). Bazel builds images as tarballs of layers.

```starlark
# backend/payment/BUILD.bazel
load("@rules_oci//oci:defs.bzl", "oci_image", "oci_tarball")
load("@rules_pkg//pkg:tar.bzl", "pkg_tar")

# 1. Put the binary in a layer
pkg_tar(
    name = "app_layer",
    srcs = [":payment_server"],
)

# 2. Assemble the image (Distroless base)
oci_image(
    name = "image",
    base = "@distroless_static",
    entrypoint = ["/payment_server"],
    tars = [":app_layer"],
    # ✨ BEST PRACTICE: Never run as root
    # Distroless provides a "nonroot" user (uid=65532) by default.
    user = "nonroot",
)
```

**Hard-Learnt Nugget:**
This image has **no OS package manager**, no shell, and no timestamp. It is immune to supply chain attacks via `apt-get` injection during build.

## **Phase 7: The `.bazelrc` (User Experience)**

**Goal:** Sane defaults for humans and CI.

### **Step 7: Configuration Layers**

```bash
# .bazelrc (Root)

# 🏛️ CONVENTION/PATTERN: Common Flags
# Print output on failure only
test --test_output=errors
# Use disk cache (preserves builds across reboots)
build --disk_cache=~/.cache/bazel-disk
# Colorful output
common --color=yes
```

💡 **TIP: CI vs. Local**
Use `--config` to separate concerns.

```bash
# .bazelrc
build:ci --noshow_progress
build:ci --remote_cache=grpcs://remotebuild.io
```

**Usage:** `bazel build //... --config=ci`

## 🔒 Security & Pitfalls

### 1. The "External" Trap

* **Pitfall:** Fetching dependencies from mutable URLs (e.g., `master` branch or `latest` tag).
* **Security:** Always use **SHA-256 integrity hashes** in `MODULE.bazel`. If the remote artifact changes by one byte, the build must fail.

### 2. Sandbox Escapes

* **Anti-Pattern:** Using `local_repository` for everything to "save time".
* **Risk:** It breaks hermeticity. Builds work on your machine but fail on CI.
* **Fix:** Trust the sandbox. If it's slow, optimize the graph, don't bypass the isolation.

### 3. Granularity Trade-offs

* **Pitfall:** Tiny targets (1 file per target) = massive graph overhead (RAM usage).
* **Pitfall:** Huge targets (monolith) = poor caching.
* **Best Practice:** Align targets with **architectural boundaries** (e.g., one Go package = one target).

## **Phase 8: Polyglot Automation (Python)**

**Goal:** Automate Python `BUILD` files and manage `pip` dependencies without manual toil.

### **Step 8: Configure the Python Extension**

In your `MODULE.bazel`, fetch the Python rules and the Gazelle plugin.

```starlark
# MODULE.bazel
bazel_dep(name = "rules_python", version = "0.36.0")
bazel_dep(name = "rules_python_gazelle_plugin", version = "0.36.0")

# Hermetic Python Toolchain
python = use_extension("@rules_python//python/extensions:python.bzl", "python")
python.toolchain(python_version = "3.11", is_default = True)

# Map pip requirements
pip = use_extension("@rules_python//python/extensions:pip.bzl", "pip")
pip.parse(
    hub_name = "pip",
    python_version = "3.11",
    requirements_lock = "//:requirements_lock.txt",
)
use_repo(pip, "pip")
```

### **Step 9: Run Python Gazelle**

Load the Python metadata into your root `BUILD` file.

```starlark
# BUILD.bazel (Root)
load("@rules_python_gazelle_plugin//:def.bzl", "GAZELLE_PYTHON_RUNTIME_DEPS")
load("@gazelle//:def.bzl", "gazelle")

# gazelle:python_root backend/api_service
# gazelle:resolve py requests @pip//requests
gazelle(
    name = "gazelle",
    data = GAZELLE_PYTHON_RUNTIME_DEPS,
)
```

**Run the generation:**

```bash
bazel run //:gazelle
```

**What You Learned:**

* ✅ Gazelle can be extended to support Python via plugins.

* ✅ `pip.parse` creates a Bazel-native repository for your libraries.

* ✅ `gazelle:resolve` maps `import requests` to `@pip//requests` automatically.

## 🚀 Summary Checklist

* ✅ **Workspace:** Defined by `MODULE.bazel` (Bzlmod).

* ✅ **Hermeticity:** Toolchains (Go, Java, C++, Python) are versioned deps.

* ✅ **Workflow:** Edit code -> `bazel run //:gazelle` -> `bazel test`.

* ✅ **Polyglot:** Use Gazelle extensions for Python/Java automation.

* ✅ **CI:** Is just `bazel test //...`.

* ✅ **Query:** You can ask questions about your architecture.
