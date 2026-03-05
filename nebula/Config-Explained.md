# Nebula config.yml Explained

> **The Anvaya:** *A well-tuned config is the difference between a high-speed mesh and a packet-dropping nightmare.*

---

## 🔑 PKI (Public Key Infrastructure)
>
> This section tells Nebula who it is and who to trust.

```yaml
pki:
  # The public part of your network's master key
  ca: /etc/nebula/ca.crt
  # This node's specific signed identity
  cert: /etc/nebula/host.crt
  # This node's private key (NEVER share this)
  key: /etc/nebula/host.key
```

---

## 🏠 Static Host Map
>
> The "Seed" list. How a node finds the Lighthouse for the first time.

```yaml
static_host_map:
  # Maps an internal Overlay IP to a public Underlay IP/Port
  "10.0.0.1": ["1.2.3.4:4242"]
```

---

## 🗼 Lighthouse
>
> Discovery configuration.

```yaml
lighthouse:
  # Am I a lighthouse? (True for the server, False for everything else)
  am_lighthouse: false
  # How long to wait before timing out a lighthouse query
  interval: 60
  # List of Overlay IPs of the lighthouses I should talk to
  hosts:
    - "10.0.0.1"
```

---

## 🌐 Network & Performance

```yaml
listen:
  # The interface to listen on (0.0.0.0 = all)
  host: 0.0.0.0
  # The UDP port. Must be open in your cloud/OS firewall.
  port: 4242
  # OS UDP buffer size. Increase to prevent packet drops under high load.
  read_buffer: 10485760  # 10MB
  write_buffer: 10485760 # 10MB
  # Number of packets to process per system call. 
  # Higher values (e.g. 64) reduce CPU usage by batching reads.
  batch: 64

tun:
  # Name of the virtual interface (default: nebula0)
  dev: nebula1
  # Dropping MTU slightly (default 1300) helps with fragmentation
  mtu: 1280

punchy:
  punchy:
    # Enables "UDP Hole Punching". Set to true for nodes behind home routers.
    punch: true
    # Keeps the hole open by sending heartbeat packets
    respond: true

# Optimization for nodes on the same LAN
# Defines which subnet is "Local"
local_range: "192.168.1.0/24"
# Tells Nebula to prefer the LAN IP over the Public IP if both are available.
# This ensures 1Gbps+ speeds for nearby nodes instead of routing via the internet.
preferred_ranges: ["192.168.1.0/24"]
```

---

## 🛰️ Relay
> Fallback for when P2P hole punching fails (e.g., Symmetric NAT). [[?](Concepts.md#relay)]

```yaml
# ON THE RELAY NODE (The Proxy)
relay:
  am_relay: true

# ON THE CLIENT NODE (The User)
relay:
  # List of Overlay IPs of nodes that have 'am_relay: true'
  relays:
    - "10.0.0.1"
  # Use relays if a direct connection cannot be established (default: true for new versions)
  use_relays: true
```

---

## 🛡️ Firewall
>
> Identity-based security rules.

```yaml
firewall:
  # Return traffic for established connections is ALWAYS allowed
  conntrack_timeout: 120s

  outbound:
    # Usually "any" to allow nodes to reach the internet
    - port: any
      proto: any
      host: any

  inbound:
    # Protocol: tcp, udp, icmp, or any
    # Port: any, 80, or 100-200
    # Identity:
    #   host: "10.0.0.5" (Single IP)
    #   groups: ["admin"] (Cryptographically signed group)
    - port: 22
      proto: tcp
      groups:
        - ssh-admins
    - port: any
      proto: icmp
      host: any # Allow ping from everyone in the mesh
```

---

## 🪵 Logging

```yaml
logging:
  # info, warning, error, or debug
  level: info
  # text or json (json is better for log aggregators)
  format: text
```
