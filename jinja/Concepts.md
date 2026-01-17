# Jinja Concepts: The Deep Dive

> **The Anvaya:** *Code generation should be predictable, not a creative process.*

---

## **<a id="expression"></a>Expression (`{{ ... }}`)**
>
> A placeholder that prints a variable's value.

**The Reality:**
Anything inside `{{ }}` is evaluated, converted to a string, and printed. This is the most basic building block.

* **Simple Variable:** `{{ user_name }}`
* **Dot Notation:** `{{ user.address.city }}`
* **Simple Math:** `{{ 1 + 1 }}` produces `2`.
* **Method Calls:** `{{ user.get_id() }}` (Only methods without arguments are allowed by default for security).

---

## **<a id="statement"></a>Statement (`{% ... %}`)**
>
> A control structure that executes logic (loops, conditionals).

**The Reality:**
Statements control the *flow* of the template generation. They do not produce direct output themselves.

* **The Rule:** If it changes the *structure* of the output (a loop, a conditional), it's a Statement. If it *prints* a value, it's an Expression.
* **Common Statements:** `{% if ... %}`, `{% for ... %}`, `{% set ... %}` (to create a variable inside the template).

---

## **<a id="filter"></a>Filter (`|`)**
>
> A function chained to an expression that transforms its value.

**The Reality:**
Filters are your data formatting toolkit.

* **Example:** `{{ "10" | int }}` converts the string "10" to the number 10.
* **With Arguments:** `{{ log_level | default('info') }}`
* **Chaining:** `{{ users | map(attribute='name') | sort | join(', ') }}` (Extract names, sort them, then join into a string).

**The Anvaya:** Use filters to keep your data model clean. The template should handle presentation, not data manipulation.

---

## **<a id="macro"></a>Macro (`{% macro ... %}`)**
>
> A reusable function within a template.

**The Reality:**
Macros are the "Don't Repeat Yourself" (DRY) principle for Jinja.

* **vs. `include`:** An `include` just pastes another file. A `macro` is a true function that can take arguments.
**Pro Feature (`caller()`):** A macro can "wrap" content, like a layout. This is common for generating HTML.

    ```jinja
    {% macro panel(title) -%}
        <div class="panel">
            <h1>{{ title }}</h1>
            <div class="content">{{ caller() }}</div>
        </div>
    {%- endmacro %}

    {% call panel("My Content") %}
        <p>This content goes inside the panel.</p>
    {% endcall %}
    ```

* **Context Note:** Auto-escaping is typically ON for web frameworks (to prevent XSS) but OFF for tools like Ansible (which generate config files). The behavior depends on the environment rendering the template.

---

## **<a id="whitespace-control"></a>Whitespace Control (`-`)**
>
> A modifier that "eats" the newline character next to a block.

**The Reality:**
Jinja templates can become messy with extra blank lines from `{% if %}` statements. The `-` modifier removes them.

* `{%- ... %}`: The hyphen *before* the block tag removes whitespace and newlines *before* it.
* `{% ... -%}`: The hyphen *after* the block tag removes whitespace and newlines *after* it.

**The Anvaya:** Use `{%- ... -%}` on almost every statement block to produce clean, human-readable output. This is the key to generating well-formatted code.

---

## **<a id="test"></a>Test (`is`)**
>
> A function chained to a variable that evaluates to `True` or `False`.

**The Reality:**
Tests are used almost exclusively inside `{% if ... %}` statements to interrogate a variable's state.

* **Existence:** `{% if user.name is defined and user.name is not none %}`
* **Math:** `{% if loop.index is divisibleby(2) %}` (for even/odd rows)
* **Type:** `{% if users is iterable and users is not string %}` (to check if you can loop over it)

**The Anvaya:** A **Filter** *changes* data. A **Test** *asks a question* about it.
