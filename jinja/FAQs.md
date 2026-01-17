# Jinja FAQs: Hard-Learnt Nuggets

> **The Anvaya:** *The edge cases define the boundaries of your knowledge.*

---

## ⚙️ Functions & Logic

**Q: Are there functions besides Macros?**
*   **The Answer:** **YES.** Jinja provides **Global Functions**.
*   **The Detail:** Unlike Macros (which you define), global functions are built-in and always available.
*   **Common Examples:**
    *   `range(5)`: Generates a list of numbers `[0, 1, 2, 3, 4]`.
    *   `dict(key='value')`: Creates a dictionary on the fly.
    *   `lipsum()`: Generates placeholder "lorem ipsum" text.
*   **The Anvaya:** Use **Macros** for repeating *your* template structure. Use **Global Functions** for generic data generation.

---

## ✍️ Escaping & Syntax

**Q: How do I print the literal `{{` characters?**
*   **The Problem:** You are generating a template *for another system* that also uses `{{ }}` (like Go templates or Angular).
*   **The Quick Fix (for a single instance):**
    *   Wrap the special character in an expression.
    *   `{{ '{{' }} my_go_variable }}` will render as `{{ my_go_variable }}`.
*   **The Robust Fix (for a large block):**
    *   Use the `{% raw %}` block. Everything inside it is treated as literal text.
    ```jinja
    {% raw %}
        <div>
            <h1>{{ title }}</h1>
            <p>{{ content }}</p>
        </div>
    {% endraw %}
    ```
    This will render exactly as written, with the `{{ ... }}` intact.
