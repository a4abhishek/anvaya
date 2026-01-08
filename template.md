# [Topic: Phased Guide Title]
> **The Anvaya:** [The core logical evolution, e.g., "Infrastructure evolves from a 'script' to a 'system'."]

---

### 🪝 The Hook
*One sentence on why "knowing the tool" isn't enough for "running production."*

### 🎯 The Important Stuffs (Roadmap)
* **Phase 1:** [Logic 1] - Minimalist deployment.
* **Phase 2:** [Logic 2] - Human-centric organization.
* **Phase 3:** [Logic 3] - Dynamic state & security.

---

### 🏗️ The Evolution (Redacted)

#### Phase 1: The Core Resource
*The absolute minimum to prove it works.*
```hcl
# ... provider config redacted ...
resource "aws_lightsail_instance" "this" {
  blueprint_id = "ubuntu_24_04" 
  bundle_id    = "nano_3_0"
}

```

#### Phase 2: The Logic of Organization

*Variables over hardcoding; outputs for visibility.*

```hcl
# ... resource config redacted ...
bundle_id = var.instance_size # <-- Logical shift to configurability

```

---

### ⚠️ Critical Pitfalls & Security

* **Anti-Pattern:** [e.g., Hardcoding AMIs].
* **The Guardrail:** [e.g., Use Data Sources].
* **Security:** [e.g., Remote state encryption].

---

### 🚀 Build On This

* [Link to official provider docs].
* **Next Step:** [Try migrating local state to S3].
