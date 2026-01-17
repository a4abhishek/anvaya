# Learning Ansible: From Zero to Production

> **The Anvaya:** *Stop writing scripts. Start defining the destination.*

## 🪝 The Hook

You manually updated the SSL cert on 9 out of 10 load balancers. The 10th one expired at midnight, taking down the payment gateway.

## **Phase 1: The Ad-Hoc Command - Remote Control**

**Goal:** Execute simple tasks across multiple servers without logging into them.

### **Step 1: The Inventory File**

Create a `hosts.ini` to define your universe.

```ini
[webservers]
192.168.1.10
192.168.1.11

[dbservers]
192.168.1.20
```

**Run a Ping:**

```bash
# -i: Inventory file
# -m: Module (ping)
# all: Target group
ansible -i hosts.ini all -m ping
```

✨ **BEST PRACTICE: SSH Keys**
Ansible relies on **Agentless** architecture. Ensure your public key (`~/.ssh/id_rsa.pub`) is on all target servers (`~/.ssh/authorized_keys`). Ansible does *not* install an agent on the target.

**What You Learned:**

* ✅ **Inventory** (server list) [[?](Concepts.md#inventory)] defines the targets.
* ✅ **Modules** (tools) [[?](Concepts.md#module)] perform the work.
* ✅ Ansible is agentless; it uses existing SSH.

**Hard-Learnt Nugget: The Static Limit**
`hosts.ini` works for physical racks. For Cloud/Autoscaling groups, static IPs are a lie. Use **Dynamic Inventory** [[?](Concepts.md#dynamic-inventory)] (e.g., `-i inventory.py`) to query the cloud API for the current "truth" at runtime.

## **Phase 2: The Playbook - Defining State**

**Goal:** Ensure a configuration exists permanently, rather than running commands once.

### **Step 2: Your First Playbook**

Scripts are imperative ("Run apt-get"). **Playbooks** (YAML state definitions) [[?](Concepts.md#playbook)] are declarative ("Ensure nginx is present").

```yaml
# site.yml
---
- name: Configure Webservers
  hosts: webservers
  become: true  # Run as root (sudo)

  tasks:
    - name: Ensure Nginx is installed
      apt:
        name: nginx
        state: present
        update_cache: true

    - name: Ensure Nginx is started
      service:
        name: nginx
        state: started
        enabled: true
```

**Run it:**

```bash
ansible-playbook -i hosts.ini site.yml
```

**The Magic of Idempotency:**
Run it twice.

* **Run 1:** Output is yellow (`changed`). Nginx installs.
* **Run 2:** Output is green (`ok`). Nginx is already there. Nothing happens.
This is **Idempotency** (safety) [[?](Concepts.md#idempotency)].

⚠️ **ANTI-PATTERN: Using `shell` module**

* **Don't:** `shell: apt-get install nginx`
* **Why:** It runs every time (not idempotent). It breaks "Check Mode" (`--check`). It relies on OS specifics.
* **Fix:** Always use the native module (`apt`, `yum`, `user`, `file`) if available.

## **Phase 3: Variables & Templates - Dynamic Config**

**Goal:** One playbook, many environments (Dev/Prod).

### **Step 3: Jinja2 Templates**

Hardcoded config files are brittle. Use Jinja2 templates.

```nginx
# templates/nginx.conf.j2
server {
    listen {{ http_port }};
    server_name {{ domain_name }};
}
```

```yaml
# site.yml (Update)
  vars:
    http_port: 80
    domain_name: "example.com"

  tasks:
    - name: Deploy Nginx Config
      template:
        src: templates/nginx.conf.j2
        dest: /etc/nginx/sites-available/default
      notify: Restart Nginx  # ⚡ Trigger Handler

  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```

**The Handler:**
The **Handler** (event-driven task) [[?](Concepts.md#handler)] `Restart Nginx` runs *only* if the template task reports a change. If config is identical, Nginx doesn't restart. Zero downtime for no-op runs.

## **Phase 4: Roles - The Library**

**Goal:** Reusability. Stop copy-pasting tasks.

### **Step 4: Directory Structure**

Move logic into a **Role** (reusable component) [[?](Concepts.md#role)].

```bash
ansible-galaxy init roles/common
```

**Structure:**

```text
roles/
  common/
    tasks/main.yml    # The logic
    handlers/main.yml # Service restarts
    templates/        # .j2 files
    vars/main.yml     # Internal variables
```

**Using Roles in Playbook:**

```yaml
# site.yml
- hosts: webservers
  roles:
    - common
    - nginx
    - security_hardening
```

🏛️ **CONVENTION/PATTERN: Role Responsibility**
A Role should do *one* thing well (e.g., "Install Postgres", "Harden SSH"). Do not create a "server_setup" role that does everything.

## **Phase 5: Secrets (Ansible Vault)**

**Goal:** Commit configuration to Git without leaking passwords.

### **Step 5: Encrypting Data**

Never store cleartext passwords. Use **Vault** (encryption) [[?](Concepts.md#vault)].

```bash
# Create encrypted file
ansible-vault create group_vars/all/secrets.yml
# Enter password...
```

**Content (`secrets.yml`):**

```yaml
db_password: "SuperSecretPassword123!"
```

**Usage in Playbook:**

```yaml
- name: Create DB User
  mysql_user:
    name: app
    password: "{{ db_password }}"
```

**Run with Vault:**

```bash
ansible-playbook site.yml --ask-vault-pass
```

**Hard-Learnt Nugget: The Alphabetical Overwrite**
When Ansible looks in a `group_vars/<group_name>/` directory, it loads YAML files in **alphabetical order**.

1. It loads `vars.yml` first.
2. It loads `vault.yml` second.
If both files define the same variable name, the one in `vault.yml` **overwrites** the one in `vars.yml` because 'v' comes after 's'.

* **Why it matters:** This implicit behavior is a common source of confusion. It's why using `vars_files` explicitly is the safer, production-grade pattern.

💡 **TIP: Explicit is Better Than Implicit**
While automatic loading is convenient, using `vars_files` in your playbook provides absolute control and visibility over the variable resolution order.

🏛️ **CONVENTION/PATTERN: The Vault Sandwich**
Always list your cleartext `vars.yml` first, followed by your encrypted `vault.yml`. This ensures your secrets correctly override your defaults.

```yaml
vars_files:
  - group_vars/kshitiz/vars.yml
  - group_vars/kshitiz/vault.yml
```

✨ **BEST PRACTICE: Vault ID**
Use a password file (`.vault_pass`) ignored by Git for CI/CD automation.
`ansible-playbook site.yml --vault-password-file .vault_pass`

## **Phase 6: Speed & Scale (Optimization)**

**Goal:** Managing 1,000 servers shouldn't take 1,000 times longer.

### **Step 6: Pipelining and Forks**

**Pipelining:**
By default, Ansible copies the module to `/tmp` via SFTP, then executes it. Pipelining executes directly over SSH (stdin), reducing connections.

```ini
# ansible.cfg
[ssh_connection]
pipelining = True
```

**Forks:**
Default is 5 parallel processes. Increase this for speed.

```ini
# ansible.cfg
[defaults]
forks = 50
```

**Hard-Learnt Nugget:**
If `pipelining = True` fails with "sudo: sorry, you must have a tty", you need to disable `requiretty` in `/etc/sudoers` on the targets.

## **Phase 7: Dynamic Logic (Facts)**

**Goal:** Write one playbook that works on both Debian and RedHat servers.

### **Step 7: Using Facts**

By default, every playbook run starts with an implicit task: **Gathering Facts**. Ansible connects to the target machine and collects hundreds of variables about its state.

**See the data:**

```yaml
- name: Debug Facts
  hosts: webservers
  tasks:
    - name: Print all available facts
      debug:
        var: ansible_facts
```

This will print a massive JSON blob containing everything from the kernel version (`ansible_kernel`) to the default IP address (`ansible_default_ipv4.address`).

**Use the data:**

The most common use is to handle different Linux distributions.

```yaml
- name: Install Common Packages
  hosts: all
  become: true
  tasks:
    - name: Install on Debian/Ubuntu
      apt:
        name: "{{ item }}"
        state: present
      with_items:
        - vim
        - htop
      when: ansible_os_family == "Debian"

    - name: Install on RedHat/CentOS
      yum:
        name: "{{ item }}"
        state: present
      with_items:
        - vim
        - htop
      when: ansible_os_family == "RedHat"
```

**What You Learned:**

* ✅ **Facts** are system variables [[?](Concepts.md#fact)] collected automatically.

* ✅ `gather_facts: true` is the default behavior.

* ✅ Use `when:` conditions with facts like `ansible_os_family` to create OS-agnostic playbooks.

**Hard-Learnt Nugget: Fact Caching**

Gathering facts can be slow on large networks. If your machine's state doesn't change often, you can cache the facts.

```ini
# ansible.cfg
[defaults]
gathering = smart
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts_cache
fact_caching_timeout = 86400  # 24 hours
```

This tells Ansible to re-use facts from a local file if they are less than a day old.

## 🔒 Security & Pitfalls

### 1. The "Root" Trap

* **Pitfall:** Logging in as root (`ansible_user: root`).

* **Security:** Create an `ansible` user with `sudo` NOPASSWD access. Disable root SSH login completely.

### 2. Check Mode Lie

* **Pitfall:** Trusting `--check` (Dry Run) blindly.

* **Risk:** Some commands (especially `shell`/`command` modules) cannot predict changes or might fail if previous steps (like installing a package) were skipped in check mode.

### 3. Committing Logs

* **Pitfall:** Committing `ansible.log` to Git.

* **Security:** Verbose logs (`-vvv`) can print sensitive data from your Vault. It's a huge security risk.

* **Fix:** Add `*.log` to your `.gitignore` file immediately.

## 🚀 Summary Checklist

* ✅ **Inventory:** Defines the *Where*.
* ✅ **Playbooks:** Define the *What* (State).
* ✅ **Roles:** Organize code for reuse.
* ✅ **Idempotency:** Safe to run repeatedly.
* ✅ **Vault:** Encrypt secrets in Git.
