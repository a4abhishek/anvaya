<img src=".github/assets/cover.png" alt="Anvaya Cover">

# Anvaya (अन्वय)

> **Anvaya** (n.): *The act of construing; re-ordering complex system chaos into a logical, personal truth.*

This is a living collection of SRE insights, pitfalls, and "magic" commands. Unlike standard documentation which is exhaustive, **Anvaya** is essentialist. It focuses only on the **Important Stuffs**—the logic that actually matters in production.

### 🧠 The Philosophy (Minimum Cognitive Load)

1. **Fast Revision:** Designed to be read and understood in under 60 seconds.
2. **Redacted Snippets:** Code blocks contain only the "trick." No boilerplate noise.
3. **Security First:** Every entry considers the security implications as a default, not an afterthought.
4. **"Almost Complete":** If a concept is here, it’s the 80% you need to actually build or fix things.

### 🛠️ Sample Entry: The "Anvaya" Format

Every file in this repo follows this precise, light-weight structure:

> #### **Terraform: The Implicit Dependency**
>
> **The Anvaya:** *Natural graph edges are always superior to manual overrides.*
>
> * **The Hook:** Why is your `terraform apply` taking 10 minutes when it should take 2?
>
> * **The Truth:** Using `depends_on` creates a sequential bottleneck. Instead, pass the > attribute directly.
>
> * **The Snippet:**
>
> ```hcl
> resource "aws_instance" "app" {
>   # ... redacted instance config ...
>   subnet_id = aws_subnet.main.id  # <-- This creates the natural edge
> }
>
> ```

* **Security Note:** Avoid hardcoding IAM ARNs; always use data sources to fetch current account context to prevent cross-account leaks.

### 📈 How to Use This Repo

* **Search:** Use `t` on the repo and search for the lession related to the technology or tools in mind.
* **Contribute:** If a snippet is too shallow, incomplete or incorrect, **refine it.**
* **Focus:** If a topic doesn't have a "Pitfall" or "Security" section, it's not finished.

### 🚀 Exploration

This repo is built to evolve. If you are looking for the "how-to" on a specific tool, check the official docs. If you are looking for the **"why it broke in production,"** you are in the right place.
