# Nebula: Hard-Learnt Nuggets

> **The Anvaya:** *A mesh network is only as strong as its weakest MTU and its most restrictive NAT.*

This document captures the "Rough Edges" and production-grade wisdom gained from real-world Nebula deployments and troubleshooting.

---

## 🏎️ Performance & The MTU Hierarchy

**The Hook:** Small `curl` requests work fine, but `git clone` or large JSON API responses hang indefinitely.

* **The Truth:** This is almost always an **MTU (Maximum Transmission Unit)** mismatch. Nebula encapsulates packets, adding a ~20-50 byte overhead.
* **The Docker Conflict:** Docker's default bridge (`docker0`) uses 1500. Nebula defaults to 1300. When a container sends a 1500-byte packet to a Nebula IP, the host drops it because it can't fit into the 1300-byte tunnel.
* **The Fix (Docker):** Align Docker with Nebula in `/etc/docker/daemon.json`:

    ```json
    { "mtu": 1300 }
    ```

* **The Cloud Quirk:** Some clouds (GCP, Azure) have an underlying MTU of 1460 or 1400. In these environments, set Nebula to **1280** to be absolutely safe.

---

## 🥊 NAT Traversal: The Symmetric Wall

**The Hook:** Your homelab node is "Online" at the Lighthouse, but you can't ping it from your AWS node.

* **The Truth:** You are likely behind a **Symmetric NAT** (common in **pfSense/OPNsense** - open-source FreeBSD-based firewall/routers, and some corporate firewalls). Symmetric NAT changes the source port for every connection, breaking UDP hole punching.
* **The Workaround:** Configure your router/firewall to use **"Static-Port"** mapping for Nebula's UDP port (4242).
* **The Nebula 1.6+ Fix:** Use a **Relay Node**. A node with a public IP can act as a "jump point" for two nodes that cannot punch holes to each other.

    ```yaml
    # relay.yml (On a public node)
    relay:
      am_relay: true
    # node.yml (On the node behind NAT)
    relay:
      use_relays: true
    ```

---

## 🔄 The Phased CA Rotation (Double CA)

**The Hook:** Your Nebula CA is expiring in 30 days. If you replace the `ca.crt` everywhere at once, the mesh will split and nodes will lose connectivity during the rollout.

* **The Strategy:** **The Double CA Trick.**
    1. Generate a **New CA**.
    2. **Phased Trust:** Update `pki.ca` on ALL nodes to include BOTH the old and new CA certificates (concatenated).
    3. **Phased Re-sign:** Issue new host certificates signed by the New CA and deploy them host-by-host. Nodes will trust each other because they still trust both CAs.
    4. **Cleanup:** Once all nodes use the New CA, remove the Old CA from the trust bundle.

---

## 🛠️ Operational Engineering

### The `nebula0` Race Condition

* **The Problem:** Ansible fails to find `ansible_nebula0` facts because the service is still starting.
* **The Anvaya:** Nebula interface creation is asynchronous. Your automation must **wait** for the interface.

    ```yaml
    - name: Wait for nebula0 interface
      wait_for:
        path: "/sys/class/net/nebula0"
        state: present
      retries: 10
      delay: 2
    ```

### LAN Speed Optimization

* **The Problem:** Two nodes on the same Wi-Fi are talking through the internet (Lighthouse) instead of the local 1Gbps LAN.
* **The Fix:** Enable `local_range` and `preferred_ranges`.

    ```yaml
    # Tell Nebula which IP range is "Local"
    local_range: "192.168.1.0/24"
    # Tell Nebula to prefer the LAN IP over the Public IP
    preferred_ranges: ["192.168.1.0/24"]
    ```

---

## ⚖️ The Developer Pain Points (What Nebula Doesn't Solve)

* **Human Roaming:** Nebula is optimized for static servers. It does not handle "Sleep/Wake" on laptops as gracefully as **Tailscale**.
* **Identity Friction:** Group membership is static. Changing a group requires a new certificate. This makes dynamic, department-based ACLs a nightmare at scale.
* **DNS Toil:** While `serve_dns` exists, it is a basic resolver. Managing a global, private split-horizon DNS remains a manual task.
* **The Alternative:** Use **Tailscale** for developers/laptops (managed control plane, SSO identity) and **Nebula** for your core infrastructure mesh (high performance, zero dependency on external vendors).
