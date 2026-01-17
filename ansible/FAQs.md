# Ansible FAQs: Hard-Learnt Nuggets

> **The Anvaya:** *If the map is wrong, the journey is doomed.*

---

## 🔐 Vault & Security

**Q: Does re-encrypting the same file change the output?**

* **The Answer:** **YES.**
* **The Reason:** Ansible Vault uses **AES-256** with a random **Salt**. Every time you run `encrypt`, a new salt is generated, producing a completely different ciphertext block.
* **The Consequence:** Git diffs are useless for vault files. You cannot look at the diff to see "what changed."
* **The Anvaya:** Trust the decrypted content, not the encrypted diff.

**Q: Can I use different passwords for Dev and Prod? (Multiple Vault IDs)**

* **The Answer:** **YES.**
* **The Feature:** Ansible supports **Multiple Vault IDs**. You can encrypt `dev_secrets.yml` with one password and `prod_secrets.yml` with another.
* **The Command:** `ansible-playbook site.yml --vault-id dev@.pass_dev --vault-id prod@prompt`
* **The Benefit:** Granular access. Developers can decrypt `dev` secrets, but only Admins have the `prod` password. You don't share the "Skeleton Key" with everyone.

---

## 🏗️ Playbooks & Execution

**Q: When should I use `shell` vs. write a Custom Module?**

* **`shell` module:**
  * **Use Case:** Quick, one-off commands, or when a command is already idempotent (`mkdir -p`).
  * **Pros:** Fast to write.
  * **Cons:** Not truly idempotent (always runs), breaks Check Mode, returns unstructured text (not data).
* **Custom Module:**
  * **Use Case:** You need to repeat the action, interact with a specific API, or manage complex state.
  * **Pros:** Idempotent, supports Check Mode, returns structured JSON data, professional and reusable.
  * **Cons:** More upfront effort (writing Python).
* **The Anvaya:** Use `shell` for prototyping. Promote it to a `Custom Module` once the logic is stable.

**Q: Why does my playbook hang at "Gathering Facts"?**

* **The Cause:** SSH connection issues or a stuck process on the target.
* **The Fix:** run with `-vvv` to see the SSH debug logs. Likely a firewall or missing Python on target.

---

## 📝 Logging & Git

**Q: Should I commit `ansible.log` to Git?**

* **The Answer:** **NO.**
* **The Risks:**
    1. **Secret Leakage:** Even with `no_log: true`, verbose runs (`-vvv`) might leak credentials.
    2. **Noise:** It changes every second. Merge conflicts will be constant.
* **The Fix:** Add `*.log` to your `.gitignore` immediately.
