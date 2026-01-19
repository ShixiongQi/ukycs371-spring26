# CloudLab Lab: Wireshark + NGINX — From “Client–Server” to Packets

## 0) What you will do

You will run an **NGINX web server** on a **single CloudLab node** and use your **laptop as the client**. While sending **low-rate** HTTP requests from your laptop to the server, you will use **Wireshark** on your laptop to capture and analyze the packets. Your goal is to connect what you see in Wireshark to the lecture concepts: **client–server**, **sockets**, **IP+port addressing**, **HTTP message format (syntax/semantics)**, and **TCP reliable transport**.

---

## 1) Learning Goals (and how you will prove each one)

By the end of this lab, you should be able to demonstrate the following with **Wireshark evidence**:

### LG1 — Client–Server Architecture

**Concept:** A client process requests a service from a server process.\
**Proof in this lab:** You will identify the **client (your laptop)** and **server (CloudLab node)** roles and show the **request → response** pattern in captured traffic.

### LG2 — Sockets & Process-to-Process Communication

**Concept:** Applications communicate using sockets; a connection is identified by a **4-tuple** (src IP, src port, dst IP, dst port).\
**Proof in this lab:** You will locate the TCP connection and record the **ephemeral client port** and **server listening port (8080)**, and show the 4-tuple in Wireshark.

### LG3 — Addressing Processes with IP + Port

**Concept:** An IP identifies a host; a port identifies a process on that host.\
**Proof in this lab:** You will explain why the server uses a fixed port while the client uses a random ephemeral port.

### LG4 — Application-Layer Protocol: HTTP Syntax & Semantics

**Concept:** HTTP defines message types, message format (headers/body), and meaning (GET, status codes, Host header, etc.).\
**Proof in this lab:** You will extract and annotate an **HTTP request line**, **headers**, and the **HTTP response status line** via “Follow TCP Stream”.

### LG5 — TCP Reliable Transport (beneath HTTP)

**Concept:** TCP provides reliable, ordered delivery; it establishes a connection before sending application data.\
**Proof in this lab:** You will capture the **3-way handshake** and explain why it must occur before HTTP data appears.

### LG6 — TLS Security: Confidentiality Changes Visibility

**Concept:** TLS encrypts application data; middleboxes/sniffers can’t read plaintext HTTP without keys.\
**Proof:** Compare HTTP vs HTTPS in Wireshark: TLS handshake is visible, but HTTP headers/body are **not** readable.

> **Policy / Platform Constraint:** This lab must remain **low-rate**. No load testing.\
> Recommended: **1 request every 2 seconds**, ~30 requests total.

---

## 2) Lab Topology (one node is enough)

* **Server:** 1 CloudLab node running **NGINX** listening on **TCP port 8080**
* **Client:** your laptop sending HTTP requests (via `curl` or browser)
* **Capture point:** Wireshark on your laptop (captures what your laptop sends/receives)

---

## 3) CloudLab Node (Server): Install & Configure NGINX (Port 8080)

> Run these on the CloudLab node.

### 3.1 Install NGINX

```bash
sudo apt-get update
sudo apt-get install -y nginx
```

### 3.2 Configure NGINX to listen on port 8080

We use 8080 to make the “port = server process address” concept explicit.

```bash
sudo sed -i 's/listen 80 default_server;/listen 8080 default_server;/' /etc/nginx/sites-available/default
sudo sed -i 's/listen \[::\]:80 default_server;/listen [::]:8080 default_server;/' /etc/nginx/sites-available/default
```

Validate + restart:

```bash
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl status nginx --no-pager
```

### 3.3 Create a recognizable response body

This helps you confirm in Wireshark that you’re looking at the right response.

```bash
echo "Hello from CloudLab NGINX - $(hostname) - $(date)" | sudo tee /var/www/html/index.html
```

### 3.4 Identify a reachable server IP

```bash
hostname -I
```

Choose the IP that your laptop can reach. Record it as:

* `SERVER_IP = _______________________`

### 3.5 Server-side sanity check (optional, but recommended)

```bash
curl -v http://127.0.0.1:8080/
```

### 3.6 Note on `SERVER_IP` (Public IP on CloudLab)

On CloudLab, the **publicly reachable IPv4 address** of a node is often an address that **starts with `128.`**. In the output of:

```bash
hostname -I
```

you may see multiple IPs. In many cases, the `128.x.x.x` one is the **public IP** that your laptop can access.

* Record this public IP as:

  `SERVER_IP = 128.___.___.___`

* You can verify it from your laptop by opening a browser and visiting:

  `http://SERVER_IP:8080/`

If the page loads (showing the “Hello from CloudLab NGINX ...” content), then you have the correct `SERVER_IP`.

---

## 4) Laptop (Client): Start Wireshark Capture

### 4.1 Choose the correct network interface

Open Wireshark and select the interface your laptop uses to reach the Internet:

* macOS: typically `en0` (Wi-Fi)
* Windows: Wi-Fi/Ethernet adapter
* Linux: `wlan0`/`eth0`/etc.

Start capture.

### 4.2 Use a display filter that targets *this* lab flow

Replace `SERVER_IP` with the actual server IP.

**Most reliable filter:**

```text
ip.addr == SERVER_IP && tcp.port == 8080
```

If Wireshark decodes HTTP on your traffic, you may also try:

```text
ip.addr == SERVER_IP && http
```

---

## 5) Laptop (Client): Send **Low-Rate** HTTP Requests

### 5.1 Recommended request pattern (30 requests, 2 seconds apart)

This is your “safe” traffic generator.

```bash
SERVER_IP=1.2.3.4
for i in $(seq 1 30); do
  curl -s -D - "http://${SERVER_IP}:8080/?seq=${i}&ts=$(date +%s)" -o /dev/null
  sleep 2
done
```

What this achieves:

* Requests include `seq` and `ts` so you can correlate packets with your intent.
* Low frequency reduces risk of CloudLab warnings.

### 5.2 Optional: Send 2 verbose requests to see more headers

```bash
SERVER_IP=1.2.3.4
curl -v "http://${SERVER_IP}:8080/test?name=cloudlab&ts=$(date +%s)"
sleep 2
curl -v "http://${SERVER_IP}:8080/"
```

---

## 6) Analysis Tasks (Each Task Maps Directly to a Learning Goal)

> For every task: you must provide **(1) evidence from Wireshark** and **(2) an explanation**.

### Task 1 — Prove LG1 (Client–Server request/response)

**Goal:** Show that the client sends a request and the server replies.

**What to do in Wireshark**

1. Apply the filter: `ip.addr == SERVER_IP && tcp.port == 8080`
2. Find one request/response pair around a given `seq=...`.

**Deliverable**

* Screenshot showing at least one **request** and the corresponding **response**
* In 2–4 sentences, explain:

  * which host is the client, which is the server
  * how you know you’re seeing a request/response exchange

---

### Task 2 — Prove LG2 + LG3 (Sockets, 4-tuple, IP+Port addressing)

**Goal:** Identify the TCP connection’s 4-tuple and explain ports.

**What to do in Wireshark**

1. Pick any packet from the flow.
2. Expand: `Internet Protocol (IP)` and `Transmission Control Protocol (TCP)`
3. Record:

   * Client IP
   * Client **source port** (ephemeral)
   * Server IP
   * Server **destination port** (8080)

**Deliverable**

* A screenshot where you can clearly see **src IP, src port, dst IP, dst port**
* Fill in the 4-tuple:

  * `(clientIP, clientPort, serverIP, 8080) = (____, ____, ____, 8080)`
* Answer in 4–7 sentences:

  * Why is the server port fixed (8080)?
  * Why does the client port look random?
  * Why is IP alone insufficient to identify the server *process*?

---

### Task 3 — Prove LG5 (TCP connection setup: 3-way handshake)

**Goal:** Show that TCP establishes a connection before application data.

**What to do in Wireshark**

1. Use this filter to find SYN packets:

```text
ip.addr == SERVER_IP && tcp.port == 8080 && tcp.flags.syn == 1
```

2. Locate the sequence:

   * SYN
   * SYN-ACK
   * ACK
3. Verify no HTTP payload appears until after handshake completes.

**Deliverable**

* Screenshot containing the SYN/SYN-ACK/ACK (can be 3 packets in view)
* Answer in 3–6 sentences:

  * Why does the handshake occur before HTTP?
  * What does TCP provide to HTTP (reliability / ordering / connection abstraction)?
  * (Optional) Identify which side initiated the connection and how you know.

---

### Task 4 — Prove LG4 (HTTP syntax & semantics via Follow TCP Stream)

**Goal:** Extract HTTP fields and explain what they mean.

**What to do in Wireshark**

1. Right-click any packet from this connection → **Follow → TCP Stream**
2. Find and label:

   * Request line: `GET /... HTTP/1.1`
   * At least **two headers** (e.g., `Host`, `User-Agent`, `Accept`)
   * Response status line: `HTTP/1.1 200 OK`
   * One response header: `Content-Length` or `Content-Type`
   * The response body (your “Hello from CloudLab…” string)

**Deliverable**

* Screenshot of the TCP Stream content
* Short write-up (6–10 sentences):

  * Identify **syntax** elements: request line, headers, blank line, body
  * Explain **semantics** of:

    * `GET` (what it requests)
    * `Host` (why needed, especially with virtual hosting)
    * status code `200` (what it indicates)

---

### Task 5 — Verify the “Low-Rate” constraint (Platform-aware behavior)

**Goal:** Demonstrate you respected the low request rate and understand why.

**What to do**

* Use Wireshark timestamps to show requests are ~2 seconds apart.

**Deliverable**

* Screenshot showing at least 3 requests with timestamps
* Answer in 3–6 sentences:

  * Why do we limit request rate on shared infrastructure like CloudLab?
  * What could happen with high-rate traffic (policy warnings, resource contention, noisy-neighbor effects)?

---

# Part B: Minimal HTTPS (TLS) Config (NGINX on 8443)

## 7) CloudLab Node: Enable HTTPS on port 8443 (Self-Signed Cert)

> Run these commands on the CloudLab node.
> Goal: `https://SERVER_IP:8443/` should work (with browser warning).

### 7.1 Install OpenSSL (if needed)

```bash
sudo apt-get update
sudo apt-get install -y openssl
```

### 7.2 Generate a self-signed certificate (valid 365 days)

```bash
sudo mkdir -p /etc/nginx/certs
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/certs/lab.key \
  -out /etc/nginx/certs/lab.crt \
  -subj "/C=US/ST=KY/L=Lexington/O=CloudLabLab/OU=CS/CN=localhost"
```

> Note: Browsers will warn because the cert is self-signed (and CN mismatch if you use an IP). This is expected for the lab.

### 7.3 Add a minimal HTTPS server block (listen 8443)

Create a new site file:

```bash
sudo tee /etc/nginx/sites-available/lab_https >/dev/null <<'EOF'
server {
    listen 8443 ssl;
    listen [::]:8443 ssl;

    ssl_certificate     /etc/nginx/certs/lab.crt;
    ssl_certificate_key /etc/nginx/certs/lab.key;

    # Minimal TLS settings (modern clients)
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers off;

    root /var/www/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF
```

Enable the site:

```bash
sudo ln -sf /etc/nginx/sites-available/lab_https /etc/nginx/sites-enabled/lab_https
```

(Optional) Put a recognizable HTTPS-only file:

```bash
echo "Hello HTTPS from CloudLab NGINX - $(hostname) - $(date)" | sudo tee /var/www/html/https.html
```

Reload NGINX:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Verify it’s listening:

```bash
sudo ss -lntp | grep 8443
```

### 7.4 Test HTTPS locally on the node

```bash
curl -vk https://127.0.0.1:8443/
curl -vk https://127.0.0.1:8443/https.html
```

---

## 8) Laptop: Test HTTPS (browser or curl)

### Option A: Browser

Visit:

* `https://SERVER_IP:8443/`

You will see a security warning (self-signed). Proceed anyway for the lab.

### Option B: curl (recommended)

```bash
SERVER_IP=128.x.x.x
curl -vk "https://${SERVER_IP}:8443/https.html"
```

---

## 9) Laptop: Wireshark Capture for HTTPS (TLS on 8443)

Use a display filter:

```text
ip.addr == SERVER_IP && tcp.port == 8443
```

You can also try:

```text
ip.addr == SERVER_IP && tls
```

### Low-rate HTTPS request generator (safe)

```bash
SERVER_IP=128.x.x.x
for i in $(seq 1 20); do
  curl -sk "https://${SERVER_IP}:8443/https.html?seq=${i}&ts=$(date +%s)" >/dev/null
  sleep 2
done
```

---

## 10) HTTPS Analysis Tasks (Prove LG6: Encryption Changes Visibility)

### Task 6 — Compare HTTP vs HTTPS visibility

**Deliverable**

* Screenshot: HTTP “Follow TCP Stream” showing readable HTTP headers/body (from port 8080)
* Screenshot: HTTPS “Follow TCP Stream” showing unreadable/encrypted payload (from port 8443)
* 4–8 sentences explaining:

  * what you can still observe with HTTPS (TCP handshake, TLS handshake metadata)
  * what you can’t observe without keys (plaintext HTTP headers and content)
  * why TLS changes this (confidentiality at/above transport)
