# Proxmox Auto-Install: answer.toml Explained

> **The Anvaya:** *This file is the DNA of your bare-metal server. Get it right, and you never have to touch a keyboard during installation again.*

This document explains the key parameters of the `answer.toml` file used by the `proxmox-auto-install-assistant`.

---

## **`[global]` Section**
>
> Defines the server's identity and core settings.

#### `country`

* **What:** Sets the country for timezone and locale defaults.
* **Example:** `country = "us"`

#### `timezone`

* **What:** The full timezone identifier.
* **Example:** `timezone = "America/New_York"`

#### `keyboard`

* **What:** The keyboard layout.
* **Example:** `keyboard = "en-us"`

#### `fqdn`

* **What:** The Fully Qualified Domain Name of the server.
* **Example:** `fqdn = "pve-node-01.brahmanda.local"`

#### `mailto`

* **What:** The email address for the `root` user, used for system notifications.
* **Example:** `mailto = "admin@example.com"`

#### `root_password`

* **What:** The password for the `root` user.
* **⚠️ CRITICAL:** This value **MUST** be a SHA-512 hashed string. Plaintext will cause the installation to fail silently.
* **How:** Generate the hash with `openssl passwd -6 "YourSecretPassword"`.
* **Example:** `root_password = "$6$rounds=656000$..."`

#### `root_ssh_keys`

* **What:** A list of public SSH keys to add to the `root` user's `authorized_keys` file.
* **✨ BEST PRACTICE:** This is the preferred way to grant initial access.
* **Syntax:** Must be an **array** of strings, even for a single key.

    ```toml
    # Correct
    root_ssh_keys = ["ssh-ed25519 AAAA... key1", "ssh-ed25519 AAAA... key2"]

    # Wrong
    # root_ssh_keys = "ssh-ed25519 AAAA... key1"
    ```

---

## **`[network]` Section**
>
> Defines the server's network configuration.

#### `source`

* **What:** How the network should be configured.
* **Values:**
  * `"from-dhcp"`: (Default) Use DHCP to get an IP address.
  * `"from-answer"`: Use the static configuration defined in this file.

#### `cidr`

* **What:** The static IP address and subnet mask in CIDR notation. Only used if `source = "from-answer"`.
* **Example:** `cidr = "192.168.68.200/24"`

#### `gateway`

* **What:** The default gateway for the server.
* **Example:** `gateway = "192.168.68.1"`

#### `dns`

* **What:** The primary DNS server.
* **Example:** `dns = "8.8.8.8"`

#### `[network.filter]`

* **What:** A sub-table to select which network interface to configure if the server has multiple.
* **Example (most common):**

    ```toml
    [network.filter]
    SUBSYSTEM = "net"
    ```

---

## **`[disk-setup]` Section**
>
> Defines how the storage will be partitioned and formatted.

#### `filesystem`

* **What:** The filesystem for the root partition.
* **Values:** `"ext4"`, `"xfs"`, or `"zfs"`.
* **Anvaya:** For a serious Proxmox host, **`zfs`** (with `raid1` or `raid10`) is the superior choice due to its robustness, snapshot capabilities, and data integrity features. `ext4` is fine for testing.

#### `disk_list`

* **What:** An array of disk identifiers to use for the installation.
* **Hard-Learnt Nugget:** Disk names are not always `sda`!
  * **SATA/SAS drives:** `["sda", "sdb"]`
  * **NVMe drives:** `["nvme0n1", "nvme1n1"]`
* **How to check:** Boot the server with a live Linux USB and run `lsblk` to see the correct disk names before you create your `answer.toml`.
