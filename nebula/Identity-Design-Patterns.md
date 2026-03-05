# Nebula: Identity-Based Design Patterns

> **The Anvaya:** *Identity is the new perimeter; firewall the role, not the address.*

---

## 🪝 The Hook

You added a 3rd worker node to your K3s cluster. You didn't have to touch a single firewall rule on your database or gateway because the new node cryptographically proved it belongs to the `worker` group.

---

## **Phase 1: Group Strategy (The Case for Sanskrit Naming)**

**Goal:** Categorize infrastructure by architectural layer rather than hostnames.

🏛️ **CONVENTION/PATTERN: Layered Identity**
In Project Brahmanda, we use functional layers to define groups:

* **`kshitiz`** (Horizon/Edge): Gateway nodes, Lighthouses, Bastions.
* **`vyom`** (Sky/Compute): General cluster nodes.
* **`vyom-control-plane`**: Specific K3s management nodes.
* **`vyom-worker`**: Scaling compute nodes.

**The Mapping:**

| Node | Role | Groups |
| :--- | :--- | :--- |
| `gateway-01` | Lighthouse | `kshitiz` |
| `k3s-master-01` | K3s API | `vyom, vyom-control-plane` |
| `k3s-node-01` | Workloads | `vyom, vyom-worker` |

---

## **Phase 2: Codifying Identity (Certificate Signing)**

**Goal:** Bake the groups into the node's "Passport."

```bash
# Signing a worker node
# -groups: These are immutable until the cert is re-issued
./nebula-cert sign \
  -name "worker-01" \
  -ip "10.42.1.211/16" \
  -groups "vyom,vyom-worker"
```

✨ **BEST PRACTICE: Multi-Group Inheritance**
Always assign a broad group (`vyom`) and a specific group (`vyom-worker`). This allows you to write "Allow all compute nodes" rules AND "Allow only masters" rules without redundancy.

---

## **Phase 3: The Segmentation Logic (Identity Firewalls)**

**Goal:** Define traffic paths based on the "Passport" groups.

**The Gateway Policy (`kshitiz`):** The gateway should be a bouncer, not a destination.

```yaml
firewall:
  inbound:
    # Allow management (SSH) ONLY from the compute cluster
    - port: 22
      proto: tcp
      groups: ["vyom"]
```

**The Compute Policy (`vyom`):** Workloads isolated from internet, open to the gateway.

```yaml
firewall:
  inbound:
    # Allow Ingress (port ranges and arrays are NOT valid; use separate rules)
    - port: 80
      proto: tcp
      groups: ["kshitiz"]
    - port: 443
      proto: tcp
      groups: ["kshitiz"]
    # Allow full inter-cluster comms
    - port: any
      proto: any
      groups: ["vyom"]
```

---

## **Phase 4: Visualizing Traffic Flow**

**Pattern A: North-South (Internet to App)**

1. **Internet** hits **Kshitiz** (Public IP).
2. **Kshitiz** forwards to **Vyom** (Overlay IP).
3. ✅ **Vyom** allows because the packet carries the `kshitiz` group signature.

**Pattern B: East-West (Inter-Node)**

1. **Worker 1** talks to **Control Plane**.
2. ✅ **Control Plane** allows because both share the `vyom` group.
3. 🚀 **Optimization:** If both are on the same Wi-Fi, they use LAN speeds via `preferred_ranges`.

---

## 🚀 Summary Checklist

* ✅ **Group Strategy:** Define roles (Edge, Compute, Storage).
* ✅ **Signing:** Bake roles into certificates.
* ✅ **Inheritance:** Use broad + specific groups.
* ✅ **Segmentation:** rules use `groups: []`, never `host: "IP"`.
