# Nebula FAQs

> **The Anvaya:** *Every "why isn't it working" question has a cert, a port, or an MTU as the answer.*

---

## 🏛️ Architecture & Theory

**Q: Does Nebula work like WebRTC?**

* **The Answer:** **Yes, conceptually.**
* **The Nuance:** Both use **UDP Hole Punching** to bypass NAT.
  * **WebRTC:** Uses STUN/ICE servers to help browsers find each other for video/audio.
  * **Nebula:** Uses the **Lighthouse** to help servers find each other for generic IP traffic.
* **The Fallback:** Just as WebRTC uses TURN servers when P2P fails, Nebula (v1.6+) uses **Relay Nodes** [[?](Concepts.md#relay)] to facilitate communication when a direct hole cannot be punched.

**Q: How is this different from a "Normal" VPN (WireGuard/OpenVPN)?**

* **Hub vs. Mesh:** Most VPNs are "Hub-and-Spoke." All your traffic from Home to AWS goes *through* a central gateway. Nebula is a **Mesh**; nodes talk directly to each other.
* **IP vs. Identity:** Traditional VPNs grant access based on your IP address. Nebula grants access based on your **Signed Identity** (Certificate Groups).
* **Routing:** In a normal VPN, you manage complex routing tables. In Nebula, every node is on a single, flat, global subnet.

---

## 🌐 Connectivity & NAT

**Q: Does all my traffic flow through the Lighthouse?**

* **The Answer:** **NO.**
* **The Reality:** The Lighthouse is only a matchmaker. Once Node A knows Node B's public IP, they talk **directly**. The Lighthouse never sees your data traffic. This is why Nebula is so fast.

**Q: My nodes can't see each other. Why?**

* **The Check:**
    1. **UDP Port 4242:** Is it open on the Lighthouse's public firewall (AWS Security Group)?
    2. **Lighthouse IP:** Did you put the correct *public* IP in the `static_host_map`?
    3. **MTU:** Large packets might be dropped. Try setting `mtu: 1280` in `config.yml`.

---

## 🔐 Security & Certificates

**Q: How do I revoke a node's access?**

* **The Problem:** Nebula certificates are valid until they expire.
* **The Solution:** You must use a **Certificate Revocation List (CRL)**. You generate a CRL containing the fingerprints of banned nodes and distribute it to all active nodes.
* **The Anvaya:** In a small mesh, just use short-lived certificates (e.g., 90 days) and re-issue them via Ansible.

**Q: What happens if the Lighthouse goes down?**

* **The Result:** Existing tunnels between nodes will **continue to work**. However, no *new* tunnels can be created until the Lighthouse is back up.
* **The Fix:** Production meshes should use **multiple Lighthouses** in different regions for high availability.

---

## 🛰️ Relays vs. Lighthouses

**Q: Is a Lighthouse automatically a Relay?**

* **The Answer:** **NO.**
* **The Difference:**
  * **Lighthouse (`am_lighthouse: true`):** The **Control Plane**. It acts as a "Directory" to help nodes find each other's public IPs. It handles metadata, not your actual data.
  * **Relay (`am_relay: true`):** The **Data Plane**. It acts as a "Proxy" for nodes that cannot connect directly (e.g., behind Symmetric NAT). It forwards your encrypted packets.
* **Best Practice:** On your public-facing "Hub" node (e.g., an AWS EC2 instance), you should usually enable **BOTH**.

    ```yaml
    lighthouse:
      am_lighthouse: true
    relay:
      am_relay: true
    ```

* **The Client Side:** Even if you have a relay, client nodes won't use it automatically. You must tell the client which Relay Overlay IP to use in its own `relay:` block.

    ```yaml
    relay:
      relays: ["10.0.0.1"] # The Overlay IP of your Relay node
      use_relays: true
    ```

**Q: Does `use_relays: true` force all traffic through the relay?**

* **The Answer:** **NO.**
* **The Logic:** Nebula is **P2P-First**. It will always attempt to punch a direct hole between nodes initially.
* **The Fallback:** `use_relays: true` (which is the default in newer versions) merely enables the *capability*. If the direct handshake fails after several attempts, Nebula will fall back to the relay to ensure connectivity.
* **Zero Penalty:** If a direct path is available, Nebula will use it. There is no performance penalty for having `use_relays: true` on a network where P2P works fine.

---

## ⚖️ Comparison & Pain Points

**Q: What are the main "Pain Points" of using Nebula?**

* **Manual Overhead:** Unlike Tailscale, there is no central dashboard. You must manually manage the CA, sign certificates for every new node, and securely distribute them. This is "Toil."
* **Static Identity:** Groups are baked into the certificate. If you want to move a node from the `web` group to the `db` group, you must **re-issue and re-deploy** a new certificate.
* **User Experience:** Nebula has no "official" GUI for end-users. It is built for SREs and servers, not for your marketing team to connect to the internal wiki.
* **Aggressive NAT:** While good at hole punching, it can still fail on complex corporate networks.

**Q: Who solves these problems better?**

* **Tailscale:**
  * **The Pro:** Zero configuration. Uses SSO (Google/GitHub) for identity. Handles key rotation and NAT traversal (DERP) flawlessly.
  * **The Con:** Proprietary control plane. You depend on their servers being up.
* **ZeroTier:**
  * **The Pro:** Layer 2 (Ethernet) mesh. Feels like a giant virtual switch. Great for gaming or legacy protocols.
  * **The Con:** More complex than Nebula/Tailscale for simple Layer 3 routing.
* **The Anvaya:** Use **Nebula** when you want a fast, free, open-source mesh for your *servers* that you fully control. Use **Tailscale** when you want your *team* to have secure access without the management headache.

---

## 🛠️ Troubleshooting Blocked Traffic

**Q: Why is SSH from Compute to Edge Gateway blocked?**

* **The Intent:** This is often intentional. Management should flow **Edge -> Compute** (Bastion pattern), not the other way around.
* **The Check:** Verify your `edge` firewall rule. If it only allows `groups: ["admin"]`, your compute nodes will be rejected.

**Q: My K3s API (6443) is blocked between workers.**

* **The Fix:** Ensure your `compute` group has an "intra-group" allow rule:

    ```yaml
    - port: 6443
      proto: tcp
      groups: ["compute"] # Allow everyone in the cluster to talk to the API
    ```

**Q: I updated a node's group in config.yml but it's not working.**

* **The Hard Truth:** Groups are **baked into the certificate**. You cannot change them in `config.yml`. You **must** re-sign a new certificate with the new `-groups` flag and restart the service.

---

## 🛠️ Operational Workflow

**Q: How do I manage 50 Nebula configs?**

* **The Answer:** **Ansible + Jinja2.**
* **The Strategy:** Use a single Jinja2 template for `config.yml`. Feed it variables (Overlay IP, Hostname, Group list) from your Ansible inventory. This ensures all nodes have identical settings for everything *except* their unique identity.
