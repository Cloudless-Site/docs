# ☁️ SSH‑First Reverse Proxy & Programmable Edge

**Cloudless** is a programmable reverse proxy and tunnel manager designed for hackers, developers, and sysadmins who believe the User Experience *is* the Protocol.

**No proprietary client is required.**
Your API is the standard OpenSSH client (`ssh`). Your identity is your SSH public key.

---

## 🗺️ The "Mental Model"

In Cloudless, the **SSH username is the command** or the **protocol**.

1.  **Control Plane:** `ssh command@cloudless.site` (e.g., `register`, `ls`, `activate`).
2.  **Data Plane:** `ssh -R ... protocol@cloudless.site` (e.g., `http`, `tcp`, `rawudp`).
3.  **Identity:** We track your **SSH Key Fingerprint (SHA256)**. No passwords, no accounts.

---

## ⚡ Quick Start: The "Proper" Way

Before exposing a production service, you typically register a stable domain name (e.g., `myapp.cloudless.site`).

### 1. Register
Reserve a subdomain linked to your SSH key.
```bash
ssh register@cloudless.site myapp.cloudless.site email=me@example.com
```
*You will receive a verification token via email (or hook).*

### 2. Verify
Prove you own the email/request.
```bash
ssh verify@cloudless.site <TOKEN_FROM_EMAIL>
```

### 3. Tunnel (Example: HTTPS)
Expose your local server running on port 8443.
```bash
ssh -R myapp.cloudless.site:443:localhost:8443 https@cloudless.site
```

Now browse:
- `https://myapp.example.com/`

---

## 🧩 Gadget Domains: The "Instant" Way

Don't want to register a permanent domain? **You don't have to.**

If you omit the hostname in your SSH tunnel request, Cloudless acts as a **Gadget Generator**. It assigns you a random, ephemeral subdomain instantly. No registration, no email, no database record.

**How to use:**
Simply leave the bind address empty (start with a colon `:`) or use `0`.

```bash
# Syntax: ssh -R :<remote_port>:<local_host>:<local_port> ...
ssh -R :80:localhost:3000 http@cloudless.site
```

**Output:**
```text
Tunnel Ready: g-x9y2z.cloudless.site -> port 80 (ACTIVE)
```

**Perfect for:** Webhooks, quick demos, and temporary file sharing.
*Note: Gadget domains are ephemeral. If you disconnect, you might lose that specific name forever.*

---

## 🚇 Tunnels: Which Protocol?

Cloudless supports different protocols. Use the right SSH username for your needs.

### 🌐 `https@` & `http@` (Web)
For web servers. Tunnels become **ACTIVE** immediately upon connection.

-   **`https@cloudless.site` (Recommended)**:
    **SNI Passthrough.** Cloudless sniffs the SNI header to route packets but **does not** terminate TLS. Encryption stays end-to-end between the visitor and your machine.
    ```bash
    ssh -R myapp.cloudless.site:443:localhost:8443 https@cloudless.site
    ```

-   **`http@cloudless.site`**:
    Cleartext HTTP. Useful if you want Cloudless to inspect traffic, modify headers via scripts, or perform logging.
    ```bash
    ssh -R myapp.cloudless.site:80:localhost:8080 http@cloudless.site
    ```

### 🔌 `tcp@` (TCP)
For databases, SSH, RDP, or custom TCP protocols.

**🛡️ Security Gate:**
To prevent port scanning and abuse, Tunnels start as **INACTIVE (Firewalled)** by default.
The **Consumer** (the person *connecting* to the exposed port) must explicitly authorize their IP address.

#### 1. Host Side (The Provider)
Run this on the machine where the service (e.g., SSH server) is running.
**You must specify the public port you want to use.**

```bash
# Expose local SSH (22) to public port 10000
ssh -R 10000:localhost:22 tcp@cloudless.site
```

#### 2. Client Side (The Consumer)
Run this on your laptop/remote machine before connecting.
```bash
# Knock to open the firewall for your current IP
ssh activate@cloudless.site
```

Notes:
- `activate@` also starts a live “watch” stream. Keep it open in a terminal.
- Activation is **not** per-client-IP allowlisting; it globally flips ACTIVE for your services.
- If you disconnect the original tunnel, the service disappears (and traffic stops).

```
# Connect to the service
ssh -p 10000 user@cloudless.site
```

---

## 🪁 UDP Tunnels: `udp@` vs `rawudp@` (Kite)

Standard SSH (`udp@`) has limitations with UDP (TCP Meltdown) because it converts packets to a stream. To serve **real UDP applications** (WireGuard, Game Servers, QUIC), you must use the **Kite** tool as a bridge.

**Reference Setup for Examples:**
*   Public Cloud Port: `10000`
*   Gateway Local Exchange (TCP): `4000`
*   IoT Target Device (UDP): `192.168.1.50:5555`

### Option A: `udp@` (Transport via SSH)
Use this if you need traffic to pass through the encrypted SSH channel (port 22) to bypass restrictive corporate firewalls.

**1. Host Side (The Provider):**
Start Kite in **SSH Adapter Mode** (no token needed), then create the SSH tunnel.

```bash
# A. Start Kite Bridge (Adapter Mode)
# Listen on Local TCP Port 4000 and forward to your UDP target
./kite -L 4000:192.168.1.50:5555

# B. Start SSH Tunnel (Forward Public 10000 -> Local 4000)
ssh -R 10000:localhost:4000 udp@cloudless.site
```

**2. Client Side (The Consumer):**
The remote user must unlock access.
```bash
# A. Activate Identity
ssh activate@cloudless.site
# Note: The activate@ command will stream live logs to confirm activation.
# Keep the terminal open or use Ctrl+C to exit; the authorization persists.

# B. Connect
nc -u cloudless.site 10000
```

### Option B: `rawudp@` (Direct **Uncrypted** High-Speed Transport)
Use this for **maximum performance** (WireGuard, Video). SSH is used only to negotiate the slot;
Kite then connects **directly** to Cloudless via a dedicated TCP stream, bypassing SSH overhead;
traffic from your machine to Cloudless and vice versa is **not encrypted**  ☣️

**1. Host Side (Reserve Slot):**
```bash
# Ask Cloudless for a raw slot on port 10000
ssh -R 10000:192.168.1.50:5555 rawudp@cloudless.site
# Output:
# > Connect String: cloudless.site:10000
# > KITE_TOKEN: A1B2-SECRET-TOKEN
```

**2. Host Side (Connect Bridge):**
Using the token from step 1:
```bash
./kite -r cloudless.site:10000:A1B2-SECRET-TOKEN -l 192.168.1.50:5555
```

**3. Client Side (The Consumer):**
As always with raw ports, the consumer must knock.
```bash
# A. Activate Identity
ssh activate@cloudless.site

# B. Connect (e.g., WireGuard Endpoint)
# Endpoint = cloudless.site:10000
```

---

## 🧠 Programmable Edge (JavaScript)

Upload **QuickJS** scripts to filter traffic, implement firewalls, or log data at the edge.
Scripts run in isolated VMs with strict memory/time budgets.

**1. Write `firewall.js`:**
```javascript
function onData(direction, buffer) {
    // direction: 0 = Ingress (Client->You), 1 = Egress (You->Client)
    if (direction === 0 && buffer.length > 5000) {
        print("Blocked jumbo packet");
        return null; // Drop packet
    }
    return buffer; // Forward packet
}
```

**2. Upload:**
```bash
cat firewall.js | ssh put@cloudless.site myapp.cloudless.site
```

**3. Download/Backup:**
```bash
ssh get@cloudless.site myapp.cloudless.site > backup.js
```

---

## ⚡ Performance & Resource Strategy ("The Guillotine")

Cloudless creates a highly efficient data plane designed to handle thousands of concurrent tunnels on modest hardware. To achieve this, strictly enforces resource discipline:

*   **Zero TIME_WAIT:** We utilize `SO_LINGER=0` (TCP Reset) to terminate connections that time out, violate protocol rules, or when the system is under heavy load.
*   **Immediate Reclamation:** Unlike standard web servers that may keep sockets in a "dying" state for up to 60 seconds, Cloudless reclaims kernel memory and file descriptors **instantly** upon closure.
*   **Client Impact:** You may occasionally see `Connection reset by peer` instead of a graceful close. This is intentional behavior to protect the infrastructure availability for all users.

---

## ⚠️  Security Notice

1. **Encryption**: Cloudless ensures encryption on the Control Plane (SSH) and offers SNI-Routing for HTTPS (end-to-end TLS).
   - For **TCP/UDP**, the transport from your machine to Cloudless is encrypted via SSH.
   - For **RAW UDP**, the transport from your machine to Cloudless is **not encrypted** ☣️ ☣️ ☣️ 
   - Traffic *from* the internet to Cloudless is cleartext (unless the application protocols like HTTPS/WireGuard are used).
   - **Do not** expose unencrypted Admin dashboards (HTTP) via TCP tunnels.

2. **DNS & Timeouts**: To verify domain ownership, Cloudless queries DNS TXT records.
   - During the `verify@` process or connecting via BYOD (Bring Your Own Domain), ensure your DNS providers respond promptly.
   - Requests pointing to stalled/unreachable DNS servers may be rejected instantly to protect the infrastructure availability.

---

## 📊 Monitoring & Dashboard

### Terminal Watch
Stream live logs for your tunnels (privacy-filtered to your fingerprint):
```bash
ssh watch@cloudless.site
```

### Web Dashboard
Generate a one-time magic link to view graphs, active sessions, and usage stats in your browser:
```bash
ssh login@cloudless.site
```

> **⚠️ Security Warning:**
> The dashboard link contains a **one-time secure session token**.
> *   Do not share screenshots of the terminal link.
> *   Do not send the link via insecure channels.
> *   Once opened, the token is consumed. To invalidate your session, simply restart your browser (if using Incognito) or wait for it to expire (24h).

---

## 🌍 Infrastructure & Lore

We take our history seriously. The location of our servers is not accidental; it is chosen to channel the energy of the giants who built the foundations of our digital world.

### 🇫🇮 Cloudless Core (Helsinki)
To guarantee maximum stability and performance, our core infrastructure is hosted approximately **25km from the University of Helsinki campus**.

This is the holy ground where the legendary **Linus Torvalds** wrote the first versions of the Linux Kernel v0.01 in 1991. We rely on the residual **magical waves** of that area for low latency and high uptime.

### 🇺🇸 The Sibling: Dynosaurus (Newark, NJ)
Cloudless has a younger sibling project called **[Dynosaurus](https://www.dynosaurus.site)**.

Unlike Cloudless, Dynosaurus is hosted near **Newark, NJ (USA)**.
Why? To stay close to the birthplace of **UNIX** (Bell Labs) and the legendary **Ken Thompson**. It, too, thrives on the historical magical waves of the Bell Labs era.

---

## 📜 Commands Reference

| Command | Arguments | Description |
| :--- | :--- | :--- |
| `register@` | `<domain> email=<addr>` | Register a new domain. |
| `verify@` | `<token>` | Verify domain ownership. |
| `release@` | `<domain>` | Delete/Release a domain. |
| `ls@` | | List your registered domains. |
| `list@` | | List currently active tunnels (RAM). |
| `status@` | | Show server global status. |
| `watch@` | | Tail live traffic logs. |
| `activate@` | | Enable your IP for TCP/UDP access. |
| `put@` | `<domain>` | Upload a JS script (stdin). |
| `get@` | `<domain>` | Download a JS script (stdout). |
| `kite@` | | Download the Kite client source and binaries. |
| `login@` | | Generate Web Dashboard link. |

---

*Cloudless is a "Hard Rock" service. We provide the raw power; you play the instruments.*
