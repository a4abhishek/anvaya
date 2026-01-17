# Learning Jinja: From Zero to Production

> **The Anvaya:** *Stop copy-pasting configuration. Generate it from a single source of truth.*

---

## 🪝 The Hook

You wrote a 50-line config file for 10 microservices. You changed a port in one, forgot to change it in another, and spent 4 hours debugging the resulting outage.

## **Phase 1: The Expression - Simple Substitution**

**Goal:** Replace a hardcoded value with a dynamic one.

### **Step 1: Your First Template**

Create a `config.j2` file. The `.j2` extension signals it's a Jinja2 template.

```nginx
# config.j2
server {
    listen {{ port }};
    server_name {{ hostname }};
}
```

**The Data (in Python/Ansible):**

```python
# data.py
context = {
    "port": 8080,
    "hostname": "api.example.com"
}
```

**The Rendered Output:**

```nginx
server {
    listen 8080;
    server_name api.example.com;
}
```

**What You Learned:**

* ✅ **Expressions** (`{{ ... }}`) [[?](Concepts.md#expression)] are placeholders for data.
* ✅ The template is just a text file. The context is your data. Rendering combines them.

## **Phase 2: Statements - Logic & Flow**

**Goal:** Generate different output based on conditions or lists.

### **Step 2: Conditionals (`if`)**

Handle optional configuration.

```nginx
# config.j2
server {
    listen {{ port }};
    server_name {{ hostname }};

    {% if enable_ssl %}
    ssl_certificate /etc/ssl/cert.pem;
    ssl_certificate_key /etc/ssl/key.pem;
    {% endif %}
}
```

**The Data:**
`{"port": 443, "hostname": "secure.example.com", "enable_ssl": True}`

**The Rendered Output:**
The `ssl_` lines are included because `enable_ssl` is true.

**What You Learned:**

* ✅ **Statements** (`{% ... %}`) [[?](Concepts.md#statement)] control the logic. They don't print anything themselves.
* ✅ `if/endif` blocks allow for conditional sections.

### **Step 3: Loops (`for`)**

Generate multiple blocks from a list.

```nginx
# config.j2
{% for server in backend_servers %}
upstream {{ server.name }} {
    server {{ server.ip }}:{{ server.port }};
}
{% endfor %}
```

**The Data:**
`{"backend_servers": [{"name": "app1", "ip": "10.0.0.1", "port": 80}, {"name": "app2", "ip": "10.0.0.2", "port": 80}]}`

**The Rendered Output:**
Two `upstream` blocks are generated, one for each item in the list.

## **Phase 3: Filters - Transforming Data**

**Goal:** Modify data at render time without changing the source.

### **Step 4: Using Filters**

**Filters** (`|`) [[?](Concepts.md#filter)] are functions chained to expressions.

```yaml
# config.j2
# Provide a default value if the variable is missing
log_level: {{ log_level | default('info') }}

# Force uppercase
app_env: {{ environment | upper }}

# Get the length of a list
server_count: {{ backend_servers | length }}
```

**Hard-Learnt Nugget:**
The `default()` filter is the most important one. Always use it for optional variables to prevent rendering errors when a variable is not defined.

## **Phase 4: Macros - Reusable Functions**

**Goal:** "Don't Repeat Yourself" (DRY) for complex blocks.

### **Step 5: Defining a Macro**

A **Macro** (`{% macro ... %}`) [[?](Concepts.md#macro)] is a function inside your template.

```nginx
# config.j2
{%- macro server_block(name, ip, port) -%}
server {{ name }} {
    address {{ ip }}:{{ port }};
    health_check /health;
}
{%- endmacro -%}

# Call the macro
{{ server_block("app1", "10.0.0.1", 8080) }}
{{ server_block("app2", "10.0.0.2", 8080) }}
```

✨ **BEST PRACTICE: Whitespace Control**
Notice the `-` inside the macro definition (`{%-` and `-%}`). This is **Whitespace Control** [[?](Concepts.md#whitespace-control)] and it "eats" the newlines around the block, producing clean output instead of messy blank lines. Use it on almost every `{% ... %}` block.

## **Phase 5: The Production Toolkit (Common Filters & Tests)**

**Goal:** Master the essential vocabulary for manipulating data.

### **Common Filters**

* `upper` / `lower`: Change string case.
* `trim`: Remove leading/trailing whitespace.
* `replace(old, new)`: Replace a substring.
* `length`: Get the count of items in a list or characters in a string.
* `sort`: Sort a list.
* `join(delimiter)`: Join a list into a single string.
* `map(attribute='...')`: Extract a property from a list of objects.
* `int` / `float`: Convert a string to a number.
* `to_json` / `from_json`: Serialize/deserialize JSON data (Ansible).
* `ipaddr`: Parse and manipulate IP addresses (Ansible).

### **Common Tests**

* `defined` / `undefined`: Check if a variable exists.
* `none`: Check if a variable is `null`.
* `iterable`: Check if you can loop over an object.
* `string`: Check if an object is a string.
* `odd` / `even`: Check a number.

---

## 🔒 Security & Pitfalls

### 1. Auto-escaping

* **Pitfall:** Rendering user-provided data directly into HTML.
* **Security:** Most Jinja environments have **auto-escaping** enabled by default. `{{ "<script>alert(1)</script>" }}` becomes `&lt;script&gt;...`.
* **The Risk:** If you disable it or use the `| safe` filter on untrusted data, you create a Cross-Site Scripting (XSS) vulnerability.

### 2. The "Undefined" Error

* **Pitfall:** `{{ my_variable }}` fails the entire render if `my_variable` isn't in the context.
* **The Fix:** Always use the `default` filter for optional variables.
  * `{{ my_variable | default(None) }}` (returns nothing if missing)
  * `{% if my_variable is defined %}` to check for existence.

---

## 🚀 Summary Checklist

* ✅ **`{{ var }}`:** Prints a variable.
* ✅ **`{% if/for %}`:** Controls logic.
* ✅ **`| filter`:** Transforms data.
* ✅ **`default()` filter:** Your best friend for preventing errors.
* ✅ **Macros:** Reusable functions for complex code blocks.
* ✅ **Whitespace Control (`-`):** Essential for clean output.
