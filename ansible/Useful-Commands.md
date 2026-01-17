# Ansible: The Magic Spells

> **The Anvaya:** *One command to rule them all, one file to bind them.*

---

## 🏗️ Ad-Hoc & Testing

**Ping All Hosts**
*   **Why:** Verify connectivity and inventory.
*   **Command:** `ansible all -m ping -i hosts.ini`

**Run Command (One-off)**
*   **Why:** Quick check (e.g., uptime) without a playbook.
*   **Command:** `ansible webservers -a "uptime" -i hosts.ini`

**Check Syntax**
*   **Why:** Validate YAML before running.
*   **Command:** `ansible-playbook site.yml --syntax-check`

**Dry Run (Check Mode)**
*   **Why:** See what *would* change without doing it.
*   **Command:** `ansible-playbook site.yml --check --diff`

---

## 🎬 Playbook Execution

**Run Playbook**
*   **Why:** The standard deploy.
*   **Command:** `ansible-playbook -i hosts.ini site.yml`

**Limit to Specific Host**
*   **Why:** Fix one broken server without touching others.
*   **Command:** `ansible-playbook site.yml --limit 192.168.1.10`

**Run Specific Tags**
*   **Why:** Run only the "nginx" parts, skip everything else.
*   **Command:** `ansible-playbook site.yml --tags "nginx"`

**Start at Task**
*   **Why:** Resume a failed run from a specific point.
*   **Command:** `ansible-playbook site.yml --start-at-task="Restart Nginx"`

---

## 🔐 Vault (Secrets)

**Encrypt File**
*   **Why:** Protect a new secrets file.
*   **Command:** `ansible-vault encrypt vars/secrets.yml`

**Edit Encrypted File**
*   **Why:** Modify secrets in place.
*   **Command:** `ansible-vault edit vars/secrets.yml`

**View Encrypted File**
*   **Why:** Read without editing.
*   **Command:** `ansible-vault view vars/secrets.yml`

**Decrypt File**
*   **Why:** Permanently remove encryption (careful!).
*   **Command:** `ansible-vault decrypt vars/secrets.yml`

---

## 🛠️ Debugging & Info

**List All Tasks**
*   **Why:** See the execution plan.
*   **Command:** `ansible-playbook site.yml --list-tasks`

**List All Hosts**
*   **Why:** Verify group membership.
*   **Command:** `ansible-playbook site.yml --list-hosts`

**Debug Variable**
*   **Why:** Print a variable value during execution.
*   **Command:**
    ```yaml
    - debug:
        var: my_variable_name
    ```
