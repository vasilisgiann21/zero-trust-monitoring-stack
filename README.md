# Zero-Trust Homelab & Secure Monitoring Infrastructure



## 🚀 Overview
This repository contains the Infrastructure as Code (IaC) for a hardened, containerized homelab environment. The objective was to deploy a robust monitoring stack (Grafana/Prometheus) alongside a network-wide DNS sinkhole (Pi-hole) utilizing a strict **Zero-Trust architecture**. 

Instead of relying on traditional perimeter security (like port-forwarding on an edge router), this infrastructure is mathematically invisible to both the public internet and the physical Local Area Network (LAN). All administrative and dashboard access is authenticated, encrypted, and routed exclusively through a Tailscale mesh VPN.

## 🏗️ Core Infrastructure Components

Every tool in this stack was chosen to solve a specific architectural problem. Here is an analytical breakdown of the stack mechanics:

### 1. Docker & Docker Compose (Containerization)
* **The Mechanic:** Instead of installing services directly onto the host Linux OS (bare-metal), all services run in isolated Docker containers. 
* **The "Why":** Installing complex software directly on the OS creates "dependency hell," where updating one package breaks another. Docker leverages Linux namespaces and cgroups to give each service its own isolated filesystem and network stack. By using `docker-compose.yml` (Infrastructure as Code), the entire server's state becomes reproducible. If the host machine suffers a catastrophic failure, this infrastructure can be rebuilt on a new machine in exactly one command, rather than spending hours configuring `apt` packages.
* **Diagram Reference:** 

### 2. Prometheus (Time Series Database)
* **The Mechanic:** Prometheus is not a standard relational database; it is a Time Series Database (TSDB) optimized for fast, high-volume metric ingestion.
* **The "Why":** Rather than waiting for services to push data to it, Prometheus uses a "pull-based" HTTP model. It routinely scrapes target endpoints (like server CPU, memory, and network I/O stats) at set intervals. It stores this data locally with multi-dimensional labels, making it incredibly efficient to query server performance at exact timestamps in the past.
* **Diagram Reference:** 

### 3. Grafana (Visualization Engine)
* **The Mechanic:** Grafana acts as the visual frontend for Prometheus. It connects to the TSDB and executes queries using PromQL (Prometheus Query Language).
* **The "Why":** Raw metrics are useless without context. Grafana translates complex, multi-dimensional time-series data into human-readable graphs, heatmaps, and alerts. It allows for real-time observability of the host OS and container health without needing to parse command-line logs.

### 4. Pi-hole (Network-Level DNS Sinkhole)
* **The Mechanic:** Pi-hole acts as the primary Domain Name System (DNS) resolver for the VPN network. When a client requests an IP address for a domain, Pi-hole cross-references the request against millions of known tracking and malware domains (Gravity Lists). 
* **The "Why":** Traditional ad-blockers run in the browser and hide elements *after* they are downloaded. Pi-hole stops the connection at the network level. If a request is malicious, Pi-hole returns `0.0.0.0` (a blackhole IP) to the client. This prevents the connection from ever leaving the router, saving bandwidth and neutralizing telemetry before it executes.
* **Diagram Reference:** 

### 5. Tailscale (Encrypted Mesh VPN)
* **The Mechanic:** Tailscale is an overlay network built on the WireGuard protocol. It uses public-key cryptography to create a direct, peer-to-peer mesh network between authorized devices.
* **The "Why":** Opening a port on a home router (NAT) exposes the server to global botnets. Tailscale uses STUN/TURN servers (DERP relays) to facilitate NAT traversal. It allows the client laptop and the server to punch a secure hole through their respective firewalls and connect directly, creating a private LAN over the public internet with zero open incoming ports.
* **Diagram Reference:** 

## 🛡️ Security Philosophy: Defense in Depth
The core engineering philosophy of this project assumes that the local Wi-Fi network is implicitly untrusted. Security is implemented in overlapping layers:

1. **Identity-Based Access:** The Tailscale WireGuard tunnel ensures the server has zero exposed ports to the public internet.
2. **Host Firewall (UFW):** The Uncomplicated Firewall acts as the OS-level gatekeeper. The default policy drops all incoming traffic, explicitly allowing only standard SSH management (Port 22) and traffic originating from the `tailscale0` encrypted interface.
3. **Strict Network Binding:** Docker bypasses host firewalls by default. To neutralize this, container ports are strictly bound to the VPN interface, ensuring services cannot broadcast to the physical network card.

## 🧠 Engineering Challenges & Analytical Resolutions

Building a split-tunnel VPN with internal DNS resolution presented several complex routing and security hurdles. 

### 1. The MTU Mismatch & Packet Fragmentation
* **The Symptom:** The Grafana web interface would endlessly load and time out over the VPN, despite ICMP pings to the server returning instantly (~1.5ms).
* **The Root Cause:** Standard Docker bridge networks use a Maximum Transmission Unit (MTU) of 1500 bytes. However, Tailscale's encrypted tunnel requires a smaller MTU of 1280 bytes due to the cryptographic overhead. Docker was pushing 1500-byte packets into a 1280-byte tunnel, causing the Linux kernel to silently drop the fragmented web traffic while allowing the tiny 64-byte ICMP pings through.
* **The Resolution:** Modified the `docker-compose.yml` network `driver_opts` to explicitly force the Docker virtual switch to use a 1280 MTU (`com.docker.network.driver.mtu: '1280'`), perfectly aligning the container network with the VPN tunnel limitations.
* **Diagram Reference:** 

### 2. The DNS Blackhole & Asymmetric Routing
* **The Symptom:** When a client connected to the VPN, internal dashboard IPs loaded correctly, but global internet traffic (e.g., YouTube) timed out completely.
* **The Root Cause:** Tailscale was injecting the Pi-hole container as the global DNS resolver. However, Pi-hole's default security posture drops queries originating from unfamiliar interfaces (like `tailscale0`). The client's OS was routing DNS requests into the tunnel, but Pi-hole was acting as a black hole, refusing to answer.
* **The Resolution:** Adjusted Pi-hole's listening behavior to "Permit all origins." Because the Pi-hole is isolated behind UFW and the VPN, it is safe to allow it to answer queries originating from the authenticated Tailscale network, restoring full internet resolution to client devices.

### 3. Linux Policy-Based Routing Conflicts
* **The Symptom:** Even after fixing DNS, the client laptop lost its default gateway route, resulting in failed pings to `8.8.8.8` while the VPN was active.
* **The Root Cause:** Tailscale utilizes Linux Policy-Based Routing (specifically Table 52) to manage traffic. A stale subnet configuration in the client's Tailscale cache was hijacking the default gateway, attempting to force all global traffic through an endpoint that was not configured for NAT routing.
* **The Resolution:** Executed a hard state reset via the Tailscale CLI (`tailscale up --reset`) to purge corrupted local routing tables, dropping conflicting subnet routes and restoring standard split-tunnel behavior.

### 4. The Docker `iptables` Firewall Bypass Loophole
* **The Symptom:** UFW was successfully enabled and configured to deny Port 3000. However, devices on the local physical Wi-Fi could still access the Grafana dashboard.
* **The Root Cause:** Docker's internal networking daemon automatically injects rules directly into the Linux kernel's `iptables` that supersede UFW. When a port is mapped globally (e.g., `"3000:3000"`), Docker physically punches a hole through the firewall to broadcast to `0.0.0.0`.
* **The Resolution:** Implemented strict interface IP binding. By mapping the port exclusively to the Tailscale IP (`"100.x.y.z:3000:3000"`), Docker is instructed to bind the listening socket *only* to the encrypted tunnel. This neutralizes the UFW bypass vulnerability and mathematically proves the container is invisible to network scanners on the local LAN.
* **Diagram Reference:** 

## 🛠️ Deployment Instructions

1.  Ensure Docker, Docker Compose, and Tailscale are installed on the host machine.
2.  Clone this repository to your host.
3.  Update the `docker-compose.yml` file:
    * Set your secure admin passwords under the `environment` sections.
    * Update the port binding with your specific Tailscale IPv4 address.
4.  Configure UFW to allow incoming traffic on `tailscale0` and `22/tcp`. 
5.  Run `sudo docker compose up -d` to instantiate the isolated network and services.
