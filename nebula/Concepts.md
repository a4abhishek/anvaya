# Nebula Concepts: The Deep Dive

> **The Anvaya:** *Nebula is not a VPN; it is a global virtual router that ignores physical geography.*

---

## **<a id="overlay-network"></a>Overlay Network**
>
> A virtual network built on top of an existing physical network.

**The Reality:**
Think of the internet as the "Underlay." Nebula creates an "Overlay" where servers in AWS, Google Cloud, and your home basement all appear to be on the same local subnet (e.g., `10.0.0.0/8`).

* **Benefit:** Traffic is encrypted end-to-end, and nodes can talk using their "Overlay IPs" regardless of their "Public IPs" or NAT situation.

---

## **<a id="lighthouse"></a>Lighthouse**
>
> A specialized Nebula node that acts as a directory for all other nodes.

**The Reality:**
Nodes don't know each other's public IPs. When Node A wants to talk to Node B, it asks the Lighthouse: "Where is Node B right now?".

* **The Workflow:**
    1. Nodes check in with the Lighthouse constantly, reporting their current public IP/Port.
    2. The Lighthouse keeps a real-time map of the network.
* **Crucial:** The Lighthouse must have a **stable, public IP** reachable by everyone. It does *not* route traffic; it only provides the map.

---

## **<a id="certificate-authority"></a>Certificate Authority (CA)**
>
> The root of trust for your Nebula network.

**The Reality:**
Nebula does not use passwords. It uses certificates.

* **The `ca.key`:** The "Master Key." You use it to sign certificates for every node. Never put this on a server; keep it on your secure laptop.
* **The `ca.crt`:** The public part. Every node must have a copy to verify other nodes.
* **Identity:** A node's certificate defines its name, its IP, and its **Groups** (used for fire-walling).

---

## **<a id="nat-traversal"></a>NAT Traversal (Punching Holes)**
>
> The ability for two nodes behind different home routers to talk directly.

**The Reality:**
Most VPNs require a central server to relay traffic. Nebula uses the Lighthouse to help nodes "punch holes" in their firewalls (UDP hole punching).

* **The Magic:** Once the Lighthouse tells Node A where Node B is, they attempt to talk **directly** to each other. If successful, traffic is peer-to-peer and extremely low latency.

---

## **<a id="groups"></a>Groups**
>
> Logical labels assigned to nodes at certificate creation.

**The Reality:**
This is "Identity-Based Networking." Instead of writing firewall rules for IP addresses, you write them for groups.

* **Example:** "Allow the `ssh-admins` group to talk to port 22 on the `production-db` group."
* **Power:** If you move a DB to a new server with a new IP, the rules don't change because the *identity* (group) remains the same.

---

## **<a id="relay"></a>Relay Node**
>
> A Nebula node with a public IP that forwards encrypted packets between nodes that cannot punch through NAT directly.

**The Reality:**
UDP hole punching fails on **Symmetric NAT** (common on pfSense, corporate firewalls). A Relay is the escape hatch — it is a "jump point" for encrypted packets, not a decryption proxy.

* **Control Plane vs. Data Plane:** A Lighthouse handles *discovery* (metadata). A Relay handles *forwarding* (data). They are different roles; you opt-in to each separately (`am_lighthouse: true`, `am_relay: true`).
* **Relay is Optional:** Nebula tries P2P first. The relay is only used if a direct tunnel cannot be established after repeated attempts.
* **Identity Check:** Relayed packets are still end-to-end encrypted. The relay node cannot read the payload.

---

## **<a id="chacha20-poly1305"></a>ChaCha20-Poly1305**
>
> An AEAD cipher used by Nebula to encrypt tunnel traffic: ChaCha20 encrypts, Poly1305 authenticates.

**The Reality:**
These two algorithms are always used together. Splitting them apart breaks the security guarantee.

* **ChaCha20** is a stream cipher — it generates a keystream from a (key + nonce), then XORs it with plaintext. Fast in pure software, constant-time by design (no timing side-channels).
* **Poly1305** is a MAC (Message Authentication Code) — it produces a 16-byte tag over the ciphertext AND any associated data (e.g., packet headers). If even one bit is tampered with in transit, decryption fails immediately and the packet is dropped.
* **AEAD** (Authenticated Encryption with Associated Data) — the combined guarantee: you get confidentiality *and* integrity in one operation, not two separate passes.
* **Why not AES-GCM?** AES-GCM requires AES-NI hardware instructions to be fast. Without them (embedded ARM, older mobile CPUs), it is 3× slower and vulnerable to timing attacks. ChaCha20-Poly1305 is equally fast everywhere — no special hardware needed.
* **Poly1305's one-time key rule:** Poly1305 requires a fresh key per message. Reusing a key with different messages completely breaks authentication. This is why it is always paired with ChaCha20 — ChaCha20 derives a fresh Poly1305 key automatically from the per-packet nonce.

**Alternatives:**

| Cipher | Notes |
|---|---|
| **AES-GCM** | Dominant alternative. Fast with AES-NI. Nonce reuse is catastrophic. |
| **XChaCha20-Poly1305** | Extended 192-bit nonce variant. Random nonce safe. Used in libsodium. |
| **AES-SIV** | Nonce-misuse resistant. Used when unique nonces are hard to guarantee. |
| **Salsa20** | ChaCha20's predecessor. Less diffusion, largely superseded. |

💡 **TIP:** ChaCha20-Poly1305 is also a first-class TLS 1.3 cipher suite (`TLS_CHACHA20_POLY1305_SHA256`). Chrome, Firefox, curl, and Go's `crypto/tls` all negotiate it when AES-NI is absent.

---

## **<a id="noise-protocol"></a>Noise Protocol**
>
> The cryptographic handshake framework, Nebula uses to establish encrypted tunnels — no CAs, no algorithm negotiation, no TLS.

**The Reality:**
Noise is not a single algorithm — it is a *framework* for building handshake protocols on top of well-audited primitives (Diffie-Hellman, ChaCha20-Poly1305, SHA256). The cipher suite is baked in at compile time; there is nothing to negotiate and therefore nothing to downgrade.

* **What Nebula uses:** The `Noise_IKpsk2` pattern — **I** = initiator's static key is known to the responder upfront, **K** = responder's static key is known to the initiator upfront, **psk2** = a pre-shared key is mixed in at message 2 (the Nebula CA certificate acts as this binding).
* **Each node proves identity** using its certificate's private key via a Diffie-Hellman operation — the private key never leaves the node.
* **Forward secrecy is mandatory** — a fresh ephemeral DH key pair is generated for every handshake. Compromising a node's long-term key doesn't decrypt past sessions.

**How the Noise Handshake works (XX pattern, for illustration):**

```
→ e                      (initiator sends ephemeral public key)
← e, ee, s, es           (responder sends ephemeral + static key, two DH operations)
→ s, se                  (initiator sends static key + final DH operation)
// Both sides have independently derived the same shared key. Done.
// Subsequent packets are encrypted with ChaCha20-Poly1305 using derived keys.
```

No round-trips spent on certificate chain downloads, OCSP checks, or cipher suite negotiation.

**Noise vs TLS Handshake:**

| | Noise (`Noise_IKpsk2`) | TLS 1.3 |
|---|---|---|
| **Algorithm negotiation** | None — fixed at compile time | Runtime (cipher suites, extensions) |
| **Authentication** | Raw static public keys, pre-shared | X.509 certificates + CA chain |
| **Certificate infrastructure** | Not needed | PKI, CAs, revocation (OCSP/CRL) |
| **Handshake size** | ~3 small UDP messages | Larger (certificate chain payload) |
| **Latency to first data** | ~1 RTT | 1 RTT (TLS 1.3), 2 RTT (TLS 1.2) |
| **Downgrade attacks** | Impossible (nothing to negotiate) | Mitigated but the surface exists |
| **Use case** | Controlled systems, P2P, known peers | Public internet, unknown servers |

💡 **TIP:** TLS is designed for the public internet where you don't know the server in advance (hence CAs). Noise is for systems where you provision keys in advance and want cryptographic minimalism. Nebula's nodes always know each other's identity via the shared CA — Noise is the right fit.

**Noise vs IKEv2/IPsec:**

| | Noise (Nebula) | IKEv2/IPsec |
|---|---|---|
| **Layer** | Application (L7) — encrypts a stream | Network (L3) — encrypts IP packets |
| **Complexity** | One spec document, fixed algorithms | ~20 RFCs, SA negotiation, rekeying |
| **Authentication** | Raw static public keys | X.509 certs or PSK |
| **NAT traversal** | Simple — works over any UDP transport | Requires NAT-T (UDP encapsulation of ESP) |
| **Kernel integration** | Userspace only | Linux `xfrm` kernel subsystem |
| **Algorithm agility** | None (no footguns) | Negotiated — many bad choices possible |
| **Implementation size** | WireGuard: ~4,000 lines (uses Noise) | Linux IPsec: ~400,000 lines |

⚠️ **ANTI-PATTERN: Confusing IKEv2/IPsec's Layer 3 scope with Nebula's scope.** IPsec tunnels *all* IP traffic between two endpoints at the kernel level — it's a network-layer VPN. Nebula is an application-layer overlay: it gives nodes a virtual IP and routes only Nebula-addressed traffic. You can run both simultaneously on the same host without conflict.
