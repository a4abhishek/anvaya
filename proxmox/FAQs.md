# Proxmox FAQs: Hard-Learnt Nuggets

> **The Anvaya:** *The defaults are for hobbyists. Production requires tuning.*

---

## Templates & Clones

**Q: Can I modify a template after creating it?**

* **The Answer:** **No.** A template is read-only and cannot be powered on.
* **The Workflow:** To update a template, you must:
    1. Create a *full clone* of the template into a new VM.
    2. Power on that new VM and make your changes (e.g., `apt upgrade`).
    3. Shut down the new VM and convert it into a *new* template (e.g., `ubuntu-24.04-v2`).
* **The Anvaya:** This enforces an immutable, versioned workflow. You don't patch templates; you replace them.

**Q: Do I need to keep the template after cloning a VM?**

* **The Answer:** **YES, absolutely.**
* **The Reason:** When you create a standard **Linked Clone**, the new VM's disk is just a thin "differencing disk." It reads all of its base operating system data from the template's disk. If you delete the template, all linked clones that depend on it will immediately break.
* **The Alternative:** A **Full Clone** copies every block from the template's disk to the new VM. It is completely independent but takes much longer to create and consumes significantly more disk space.

**Q: Should templates be created with minimal resources?**

* **The Answer:** **Yes, absolutely.** You can safely define a template with 1 core and 1GB RAM.
* **The Reason:** The resources defined on the template are just a set of defaults. They are irrelevant because the template is never powered on, and tools like Terraform will override them at clone time. Keeping them minimal makes the template's purpose as a "base image" clear.

**Q: If my template has 2 cores, will all my clones have 2 cores?**

* **The Answer:** **No.** The resources of a cloned VM are determined at creation time, not by the template's defaults.
* **The Workflow (Terraform):** Your Terraform code is the source of truth. It overrides the template's settings.

    ```hcl
    # main.tf
    resource "proxmox_vm_qemu" "k3s_worker_node" {
      clone = "ubuntu-2404-template" // The base image

      # --- Terraform OVERRIDES the template's defaults ---
      cores   = 4      // We want 4 cores, not the template's 1
      memory  = 16384  // We want 16GB RAM, not the template's 1GB
      
      disk {
        size = "128G" // We want a 128GB disk, not the template's 16GB
        ...
      }
    }
    ```

## ⚙️ Performance

**Q: `qcow2` vs `raw`? Which disk image format should I use?**

* **The Answer:** It depends entirely on your **underlying storage system**.
* **`qcow2` (QEMU Copy-On-Write):**
  * **Features:** Supports thin provisioning (grows on demand) and internal snapshots at the file level.
  * **Performance:** Has a slight performance overhead due to its metadata layer.
  * **When to Use:** On simple, file-level storage like **NFS** or a local directory on an **ext4** filesystem. Here, you *need* `qcow2` to provide features the underlying storage lacks.
* **`raw` (Raw Disk Image):**
  * **Features:** A bit-for-bit image of a disk. It has no special features of its own.
  * **Performance:** **Maximum performance.** There is no metadata overhead.
  * **When to Use:** On advanced storage that provides its own features, like **ZFS** or **LVM-Thin**. These systems handle snapshots and thin provisioning at the block level, which is far more efficient. Using `qcow2` on ZFS adds a redundant, performance-killing layer.
* **The Anvaya:** Match the disk format to the storage. If your storage is "smart" (ZFS), use a "dumb" disk (`raw`). If your storage is "dumb" (a simple directory), use a "smart" disk (`qcow2`).

**Q: Why is my disk I/O slow?**

* **The Cause:** You are likely using the default `qcow2` file format on a standard `ext4` filesystem.
* **The Fix (for performance):** Use ZFS as your storage backend and provision `raw` disks for your VMs. `raw` on ZFS provides near-native performance because it bypasses the overhead of the `qcow2` format.
* **The Trade-off:** `qcow2` supports thin provisioning and snapshots at the file level. With `raw` on ZFS, these features are handled by ZFS itself, which is more robust.
