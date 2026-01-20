# Proxmox: Golden Image Checklist

> **The Anvaya:** *A perfect template is generic in content but specific in purpose.*

## Goal

This checklist provides a repeatable process for creating a "golden image" template from a cloud image, ready to be cloned by Infrastructure as Code tools like Terraform.

This document is a **curated checklist**, not just a list of commands. Its value lies in the "Hard-Learnt Nuggets" and "Best Practices" that explain the *why* behind critical but non-obvious steps. When updating, maintain this focus on explaining the rationale.

---

## **Part 1: Prepare Your Keys (Local Machine)**

**Goal:** Securely retrieve the SSH key that will be baked into the template for emergency/initial access.

1. **Retrieve Keys:** Get your master public (`id_master.pub`) and private (`id_master`) keys from your password manager (e.g., 1Password).

    ```bash
    # Example using 1Password CLI
    op read "op://YourVault/MasterKey/public key" > /tmp/id_master.pub
    op read "op://YourVault/MasterKey/private key" > /tmp/id_master
    chmod 600 /tmp/id_master
    ```

2. **Upload to Proxmox:** Copy the key pair to the Proxmox host. They will be used to configure the template and for the initial SSH connection.

    ```bash
    scp /tmp/id_master* root@proxmox.host:/tmp/
    ```

## **Part 2: Create & Configure VM (Proxmox Host)**

**Goal:** Create a temporary VM from a cloud image and inject the necessary boot-time configuration.

Connect to your Proxmox host (`ssh root@proxmox.host`) and run the following commands.

1. **Create the VM:** Define a VM with minimal resources. It's just a blueprint.

    ```bash
    # Purge any previous version
    qm stop 9000 && qm destroy 9000 --purge
    
    # Create the base VM
    qm create 9000 --name "ubuntu-2404-template" --memory 1024 --cores 1 --net0 virtio,bridge=vmbr0
    ```

2. **Import the Cloud Image:**
    *This assumes you have already downloaded the cloud image to `/var/lib/vz/template/qcow/`.*

    ```bash
    qm importdisk 9000 /var/lib/vz/template/qcow/noble-server-cloudimg-amd64.qcow2 local-lvm
    ```

3. **Configure Hardware & Boot:**

    ```bash
    qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0 --ide2 local-lvm:cloudinit --boot c --bootdisk scsi0 --serial0 socket --vga serial0
    ```

4. **Inject Boot-Time Configuration:**

    > 💡 **Hard-Learnt Nuggets:** Attach a static IP<br>
    > The new VM doesn't have the guest agent yet, so Proxmox can't see its DHCP IP.
    > We force a temporary, known IP so we can SSH into it.

    ```bash
    qm set 9000 --ipconfig0 ip=192.168.1.99/24,gw=192.168.1.1
    ```

    > ✨ **Best Practice:** Inject the SSH key<br>
    > This is how you gain access to a cloud-init image without a password.

    ```bash
    qm set 9000 --sshkeys /tmp/id_master.pub
    ```

## **Part 3: Install Guest Agent (Inside VM)**

**Goal:** Install the `qemu-guest-agent` inside the VM to enable full communication with the Proxmox hypervisor.

1. **Start the VM:**

    ```bash
    qm start 9000
    ```

2. **Connect to the VM:** From your local machine, use the private key and the temporary static IP.

    ```bash
    # Wait ~60 seconds for the VM to boot.
    # If you get a "host key has changed" warning, remove the old key:
    # ssh-keygen -R 192.168.1.99
    ssh -i /tmp/id_master ubuntu@192.168.1.99
    ```

3. **Install Agent & Update:** Inside the VM, run:

    ```bash
    sudo apt-get update && sudo apt-get upgrade -y
    sudo apt-get install -y qemu-guest-agent
    ```

    > You may see a message that the service is `disabled or a static unit`. This is expected. It will be enabled on the clones.

4. **Shutdown the VM:**

    ```bash
    sudo shutdown -h now
    ```

---

## **Part 4: Finalize Template (Proxmox Host)**

**Goal:** Reset temporary settings and convert the VM into a read-only template.

1. **Reset IP Configuration:** Set the network back to DHCP so that cloned VMs can get their own unique IP addresses.

    ```bash
    qm set 9000 --ipconfig0 ip=dhcp
    ```

2. **Convert to Template:** This makes the VM read-only and ready for cloning.

    ```bash
    qm template 9000
    ```

3. **Clean Up Keys:** Remove the temporary key files from the Proxmox host and your local machine.

    ```bash
    # On Proxmox host
    rm /tmp/id_master*
    
    # On local machine
    rm /tmp/id_master*
    ```

✅ **Verification:** In the Proxmox UI, you will now see a new template icon for VM 9000. It is ready to be used as a `clone` source in your Terraform or Ansible automation.
