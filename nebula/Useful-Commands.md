# Nebula: Useful Commands

> **The Anvaya:** *Certificates are your only identity. Manage them with care.*

---

## 🔑 Certificate Management

**Create CA**

* **Why:** Generate the root of trust for your mesh.
* **Command:** `./nebula-cert ca -name "MyMeshName"`

**Sign a Node Certificate**

* **Why:** Issue an identity to a new server or laptop.
* **Command:** `./nebula-cert sign -name "hostname" -ip "10.0.0.5/24" -groups "admin,web"`

**Verify a Certificate**

* **Why:** Inspect the contents (IP, name, groups) of a `.crt` file.
* **Command:** `./nebula-cert print -path ./host.crt`

---

## 🚀 Execution & Maintenance

**Run Nebula (Foreground)**

* **Why:** Test your configuration and see real-time logs.
* **Command:** `sudo ./nebula -config config.yml`

**Validate Config**

* **Why:** Check YAML syntax and logic before starting service.
* **Command:** `./nebula -config config.yml -test`

**Follow Service Logs**

* **Why:** Tail live Nebula output when running as a systemd service.
* **Command:** `journalctl -u nebula -f`

---

## 🐞 Troubleshooting

**List Active Tunnels**

* **Why:** See which peers are currently connected.
* **Command:** `sudo kill -SIGUSR1 $(pgrep nebula)` (Dumps tunnel list to the **Nebula log** — check `journalctl -u nebula` or stdout if running in foreground).

**Graph Connections**

* **Why:** Visualize the mesh connections.
* **Command:** `sudo kill -SIGUSR2 $(pgrep nebula)` (Dumps a JSON graph of connections to the **Nebula log**; redirect with `-log-format json` for parsing).

**Check Interface**

* **Why:** Verify the overlay IP is assigned.
* **Command:** `ip addr show nebula0`

---

## ⚙️ Installation (Linux)

**Create Systemd Service**

* **Why:** Ensure Nebula starts on boot.
* **Command:**

    ```bash
    sudo tee /etc/systemd/system/nebula.service <<EOF
    [Unit]
    Description=Nebula Mesh Network
    After=network.target

    [Service]
    ExecStart=/usr/local/bin/nebula -config /etc/nebula/config.yml
    Restart=always

    [Install]
    WantedBy=multi-user.target
    EOF
    sudo systemctl enable --now nebula
    ```

---

## 📡 NetOps Debugging

**Verify UDP Heartbeats**

* **Why:** Check if packets are actually arriving on port 4242.
* **Command:** `sudo tcpdump -ni any udp port 4242`

**Check Process Listening**

* **Why:** Confirm the binary has successfully bound to the port.
* **Command:** `sudo netstat -ulnp | grep nebula`

**Test Path MTU**

* **Why:** Debug "SSH hangs" or "Large files fail".
* **Command:** `ping -M do -s 1272 10.0.0.1` (Test if 1300-byte packets fit).
