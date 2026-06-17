# Hybrid Architecture HA K3s — 3 Masters × 5 Workers (Local + Cloud)

This hybrid plan won't work. See [TRAPS.md](./TRAPS.md) for reasons.

## Hardware

Each host runs **2 K3s containers** (1 master + 1 worker) using **Docker** — no VMs, no overhead. The Pi is the bootstrap node. Plus 2 Aliyun cloud workers for a total of **8 nodes** (3 masters + 5 workers).

| Device                             | Arch  | CPU |    RAM | Scope  | Master Node                     | Worker Node                     |
| ---------------------------------- | ----- | --: | -----: | ------ | ------------------------------- | ------------------------------- |
| Raspberry Pi 5 (`rp5-01`)          | ARM64 |  4C |   8 GB | Local  | `k3s-m1` (ARM64, 192.168.x.129) | `k3s-w1` (ARM64, 192.168.x.129) |
| MBA M2 (`mbp-m2`) — macOS 15       | ARM64 | 10C | ~16 GB | Local  | `k3s-m2` (ARM64, 192.168.x.206) | `k3s-w2` (ARM64, 192.168.x.206) |
| PC i7 (`pc-i7`, WSL2) — Windows 11 | AMD64 |  8C | ~32 GB | Local  | `k3s-m3` (AMD64, 192.168.x.131) | `k3s-w3` (AMD64, 192.168.x.131) |
| Aliyun VM 1 (`aliyun-vm-1`)        | AMD64 |   — |      — | Public | —                               | `k3s-w4` (AMD64, 47.104.x.219)  |
| Aliyun VM 2 (`aliyun-vm-2`)        | AMD64 |   — |      — | Public | —                               | `k3s-w5` (AMD64, 8.159.x.62)    |

> **Fault tolerance**: 3 etcd masters provide quorum as long as ≥2 are online. Loss of any **1 local host** (1 master + 1 worker) is tolerated. Loss of a **public node** (worker only) has no impact on control plane. Worst case: 1 local host + all public nodes down → 2 masters still have quorum, cluster operates normally.

> **Energy strategy**: ARM64 workers run 24/7 — they're power-efficient (~5–15W). AMD64 workers are **on-demand standby** — stopped by default, started only for AMD64-only workloads or when cluster needs extra capacity.

## Architecture Strategy

- **Scale**: 3 masters (embedded etcd) + 5 workers = 8-node cluster across 5 physical hosts.
- **Mixed-arch**: 1× ARM64 worker (w1, always-on) + 4× AMD64 workers (w2, w3, w4, w5, on-demand).
- **Multi-arch images**: All container images should build/pull for both `linux/amd64` and `linux/arm64` (Docker buildx). Schedule with `nodeSelector` or `nodeAffinity`.
- **Control plane**: K3s masters use **embedded etcd** (built-in, no external DB).
- **Energy-aware**: Default workloads land on ARM64 workers. AMD64 workers are activated on demand for arch-specific or resource-heavy jobs.

---

## Approach: Docker K3s (No VMs, No k3d)

All nodes run as `rancher/k3s` Docker containers. Networking differs per platform:

- **Pi & MBA**: `--network host` — etcd peers use real LAN IPs natively.
- **PC i7 (WSL2)**: Port mapping (`-p`) — required because Docker Desktop WSL2 doesn't expose host-network ports to LAN.

The Pi is the bootstrap node (native Docker on Linux avoids all WSL2 networking quirks).

### Docker confirmed on all hosts

| Host                        | Runtime                        | Version |
| --------------------------- | ------------------------------ | :-----: |
| Raspberry Pi 5 (`rp5-01`)   | Docker CE (native)             | v29.5.3 |
| MBA M2 (`mbp-m2`)           | Docker Desktop (Apple Silicon) | v29.5.3 |
| PC i7 (`pc-i7`)             | Docker in WSL2                 | v29.5.3 |
| Aliyun VM 1 (`aliyun-vm-1`) | Docker CE (Debian)             | v29.5.3 |
| Aliyun VM 2 (`aliyun-vm-2`) | Docker CE (Debian)             | v29.5.3 |

---

## Network Topology

```text
┌─────────────────────────────────────────────────┐
│                   LAN                           │
│             192.168.x.0/24                     │
│                                                 │
│  ┌──── RP5 ────┐ ┌──── PC i7 ──┐ ┌──── MBA ────┐ │
│  │ host: .129  │ │ host: .131  │ │ host: .206  │ │
│  │k3s-m1 (srv*)│ │ k3s-m3 (srv)│ │ k3s-m2 (srv)│ │
│  │ k3s-w1 (agt)│ │ k3s-w3 (agt)│ │ k3s-w2 (agt)│ │
│  └─────────────┘ └─────────────┘ └─────────────┘ │
├─────────────────────────────────────────────────┤
│                      Cloud                       │
│                                                 │
│     ┌── k3s-w4 ──┐        ┌── k3s-w5 ──┐       │
│     │ 47.104.x   │        │ 8.159.x    │       │
│     │ 172.31.x   │        │ 172.17.x   │       │
│     └────────────┘        └────────────┘       │
│            ↕ Tailscale / WireGuard ↕            │
└─────────────────────────────────────────────────┘

  * k3s-m1 is the **bootstrap node** (Pi, native Docker `--network host`).
```

- **Container networking**: Differs per platform:
  - **Pi & MBA**: `--network host` — containers share the host's network stack. Works on native Linux and macOS.
  - **PC i7 (WSL2)**: Port mapping (`-p 6443:6443 -p 2380:2380`) — `--network host` doesn't expose ports to LAN on Docker Desktop WSL2 backend.
- **Lan hosts**: Communicate over WiFi directly via `192.168.x.0/24`.
- **Cloud hosts**: On separate networks — cannot reach LAN IPs directly. Need an overlay (Tailscale or WireGuard).
- **LAN details**:
  - **PC i7**: WiFi-only (`192.168.x.131`), no Ethernet.
  - **Pi dual-homed**: `eth0` = `192.168.x.3` (wired), `wlan0` = `192.168.x.129` (WiFi, cluster subnet).
  - **MBA M2**: WiFi-only via `en0` at `192.168.x.206`.
  - All on the same router (`192.168.x.1`). No inter-subnet routing needed.
- **mDNS / avahi**: Optional for `.local` hostname resolution.

---

## Step-by-Step Setup

### Phase 0: Docker Prerequisites on all servers

```bash
docker version
```

---

### Phase 1: Bootstrap k3s-m1 on Raspberry Pi 5

Pi runs native Linux — `--network host` works correctly, no WSL2 quirks.

```bash
# Bootstrap the cluster
docker run -d --privileged --name k3s-m1 \
  --restart unless-stopped \
  --network host \
  -e K3S_CLUSTER_INIT=true \
  -e K3S_KUBECONFIG_MODE=644 \
  rancher/k3s:v1.36.1-k3s1 server
```

> `K3S_CLUSTER_INIT=true` enables embedded etcd. `--network host` shares the Pi's network — etcd advertises `192.168.x.129` directly.

After creation, get the token:

```bash
docker exec k3s-m1 cat /var/lib/rancher/k3s/server/node-token
# → K10...<token>
```

---

### Phase 2: Join MBA M2 as Master

MBA uses Docker Desktop (macOS) — `--network host` works here too.

```bash
docker run -d --privileged --name k3s-m2 \
  --restart unless-stopped \
  --network host \
  -e K3S_TOKEN=<token> \
  -e K3S_KUBECONFIG_MODE=644 \
  rancher/k3s:v1.36.1-k3s1 server \
  --server https://192.168.x.129:6443
```

---

### Phase 3: Join PC i7 as Master

PC i7 runs Docker Desktop with WSL2 — `--network host` doesn't expose ports to LAN. Use port mapping instead.

```powershell
docker run -d --privileged --name k3s-m3 `
  --restart unless-stopped `
  -e K3S_TOKEN=<token> `
  -e K3S_KUBECONFIG_MODE=644 `
  rancher/k3s:v1.36.1-k3s1 server `
  --server https://192.168.x.129:6443
```

---

### Phase 4: Join Workers

On **each host**, run a worker (agent) container. Workers connect outbound — they don't need open inbound ports.

**Pi** (native Linux — `--network host`):

```bash
docker run -d --privileged --name k3s-w1 \
  --restart unless-stopped \
  --network host \
  -e K3S_URL=https://192.168.x.129:6443 \
  -e K3S_TOKEN=<token> \
  rancher/k3s:v1.36.1-k3s1 agent
```

**MBA M2** (macOS — `--network host`):

```bash
docker run -d --privileged --name k3s-w2 \
  --restart unless-stopped \
  --network host \
  -e K3S_URL=https://192.168.x.129:6443 \
  -e K3S_TOKEN=<token> \
  rancher/k3s:v1.36.1-k3s1 agent
```

**PC i7** (WSL2 — port mapping):

```powershell
docker run -d --privileged --name k3s-w3 `
  --restart unless-stopped `
  -p 30443:6443 `
  -e K3S_URL=https://192.168.x.129:6443 `
  -e K3S_TOKEN=<token> `
  rancher/k3s:v1.36.1-k3s1 agent
```

---

### Phase 5: Verify Cluster

From any host with access to the cluster:

```bash
# Copy kubeconfig from the Pi bootstrap container
docker cp k3s-m1:/etc/rancher/k3s/k3s.yaml ~/.kube/config

# Edit and change server IP to 192.168.x.129 (if needed)

# Verify nodes
kubectl get nodes -o wide

# Should see:
# NAME     STATUS   ROLES                       AGE   VERSION   INTERNAL-IP
# k3s-m1   Ready    Control-Plane,etcd,Master   ...   v1.36.1   192.168.x.129
# k3s-m2   Ready    Control-Plane,etcd,Master   ...   v1.36.1   192.168.x.206
# k3s-m3   Ready    Control-Plane,etcd,Master   ...   v1.36.1   192.168.x.131
# k3s-w1   Ready    <none>                      ...   v1.36.1   192.168.x.129
# k3s-w2   Ready    <none>                      ...   v1.36.1   192.168.x.206
# k3s-w3   Ready    <none>                      ...   v1.36.1   192.168.x.131
```

---

## Post-Setup Considerations

### 1. Multi-Arch Workload Scheduling

Since workers span ARM64 (w1) and AMD64 (w2, w3, w4, w5), use nodeSelector or nodeAffinity:

```yaml
# For ARM64 images
nodeSelector:
  kubernetes.io/arch: arm64

# For AMD64 images
nodeSelector:
  kubernetes.io/arch: amd64
```

For automatic scheduling, use a multi-arch image registry and let containerd pick the right manifest.

### 2. Storage

- **Local storage**: `local-path-provisioner` (built into K3s) for single-node workloads
- **Distributed storage** (recommended for HA): Install [Longhorn](https://longhorn.io/) — it works on mixed-arch clusters.
- **NFS**: If a NAS is available, use the NFS CSI driver.

### 3. Load Balancing & Ingress

| Component          | Recommendation                                                             |
| ------------------ | -------------------------------------------------------------------------- |
| Ingress Controller | **Traefik** (built into K3s, port 80/443 exposed on all nodes)             |
| Load Balancer (L4) | **MetalLB** (layer 2 mode) — assigns VIPs to services of type LoadBalancer |
| DNS                | ExternalDNS + `dnsmasq` on router, or `nip.io`/`sslip.io` for dev          |

### 4. Observability

| Layer      | Tool                            |
| ---------- | ------------------------------- |
| Metrics    | Prometheus + kube-state-metrics |
| Dashboards | Grafana                         |
| Logging    | Loki + Promtail (or Fluentbit)  |
| K3s UI     | Kube-explorer / Octant          |

### 6. On-Demand AMD64 Worker Activation

AMD64 workers (w3, w4, w5) are **stopped by default** to save energy. Start them when needed:

```bash
# PC i7 worker (k3s-w3)
docker start k3s-w3

# Aliyun workers (w4, w5) — start their Docker containers:
docker start k3s-w4  # aliyun-vm-1
docker start k3s-w5  # aliyun-vm-2
```

To verify which workers are online:

```bash
kubectl get nodes -l node-role.kubernetes.io/agent=true
```

To stop them after use:

```bash
docker stop k3s-w4
docker stop k3s-w5
```

> **Note**: The Aliyun VMs are billable by the hour — stopping the containers alone doesn't stop the VM billing. Shut down the entire VM if you want to save cloud costs.

## Adding the Public Nodes (Aliyun)

### Node 1: `aliyun-vm-1`

| Detail     | Value                |
| ---------- | -------------------- |
| Hostname   | `aliyun-vm-1`        |
| OS         | Debian 13            |
| Arch       | AMD64                |
| Private IP | `172.31.x.171` (VPC) |
| Public IP  | `47.104.x.219` (NAT) |
| Docker     | ✅ v29.5.3           |
| Node       | `k3s-w4` (worker)    |

### Node 2: `aliyun-vm-2`

| Detail     | Value               |
| ---------- | ------------------- |
| Hostname   | `aliyun-vm-2`       |
| OS         | Debian 13           |
| Arch       | AMD64               |
| Private IP | `172.17.x.39` (VPC) |
| Public IP  | `8.159.x.62`        |
| Docker     | ✅ v29.5.3          |
| Node       | `k3s-w5` (worker)   |

The public nodes cannot reach LAN IPs directly — they are on entirely separate networks. The cluster API and node communication need an overlay.

### Option A: Tailscale (Recommended)

Install Tailscale on **all 5 hosts**. Each gets a `100.x.x.x` Tailscale IP. Then **delete the old bootstrap container** and recreate with the Tailscale IP:

```powershell
# On PC i7 — recreate bootstrap with Tailscale IP
docker rm -f k3s-m3
docker run -d --privileged --name k3s-m3 `
  --restart unless-stopped `
  --network-host `
  -e K3S_CLUSTER_INIT=true `
  -e K3S_KUBECONFIG_MODE=644 `
  rancher/k3s:v1.36.1-k3s1 server
```

On each Aliyun node:

```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# Join as worker via Tailscale
# On aliyun-vm-1:
docker run -d --privileged --name k3s-w4 \
  --restart unless-stopped --network host \
  -e K3S_URL=https://100.x.x.x:6443 \
  -e K3S_TOKEN=<token> \
  rancher/k3s:v1.36.1-k3s1 agent

# On aliyun-vm-2:
docker run -d --privileged --name k3s-w5 \
  --restart unless-stopped --network host \
  -e K3S_URL=https://100.x.x.x:6443 \
  -e K3S_TOKEN=<token> \
  rancher/k3s:v1.36.1-k3s1 agent
```

> Both Aliyun nodes are **AMD64** — workloads without arch restrictions can schedule on any of the 3 AMD64 workers (w3, w4, w5).

### Option B: WireGuard

Set up a WireGuard tunnel between the Aliyun VM and one LAN host (e.g., the i7). The cluster API then becomes reachable at the WireGuard tunnel IP.

### Option C: Public Port Exposure (Not Recommended)

Open port 6443 on the i7's firewall and point the Aliyun node at it. Risk: your k3s API is internet-facing. Only viable with strong IP whitelisting on the Aliyun side.

---
