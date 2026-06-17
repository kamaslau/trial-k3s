# Pure Cloud K3s — 3 Masters × Built-in Workers (3 VPS)

## Hardware

| Node  | Role              | Arch  | Spec         |   Disk | Hostname      |
| ----- | ----------------- | :---: | ------------ | -----: | ------------- |
| VPS 1 | Server (w/ agent) | AMD64 | **2C, 3 GB** |  79 GB | `*-aliyun-00` |
| VPS 2 | Server (w/ agent) | AMD64 | **2C, 2 GB** |  40 GB | `*-aliyun-01` |
| VPS 3 | Server (w/ agent) | ARM64 | **4C, 8 GB** | 117 GB | `*-raspi-00`  |

All nodes run Debian + Docker, connected via Tailscale. `--network host` works natively — no port mapping needed.

## Architecture

- **Masters**: All 3 VPS run as servers with embedded etcd. Tolerates loss of any 1 node.
- **Workers**: Built-in — K3s server automatically runs a local worker, no separate agent containers needed.
- **Networking**: Over Tailscale — each node uses its `100.x.x.x` IP for direct, NAT-free communication.

## Setup

### Phase 0: Prerequisites

```bash
# 1. Check hardware specs
echo "CPU: $(nproc) C, RAM: $(awk '/MemTotal/{printf "%.0f GB", $2/1024/1024}' /proc/meminfo), Disk: $(df -h / | awk 'NR==2{print $2}'), Docker: $(docker -v | awk '{print $3}' | tr -d ','), OS: $(. /etc/os-release && echo "$ID $VERSION_ID"), Arch: $(case $(uname -m) in x86_64) echo amd64;; aarch64) echo arm64;; *) uname -m;; esac)"
# Should see:
# CPU: 2C, RAM: 2 GB, Disk: 40G, Docker: 29.5.3, OS: debian 13, Arch: amd64

# 2. Install Tailscale (run on ALL nodes)
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# Note each node's Tailscale IP for later
TS_IP=$(ip addr show tailscale0 | awk '/inet /{print $2}' | cut -d/ -f1)
echo "Tailscale IP: $TS_IP"
# Should see:
# Tailscale IP: 100.x.x.x
```

### Phase 1: Bootstrap on VPS 1

```bash
# Start the first master with TLS SAN for public IP
TS_IP=$(ip addr show tailscale0 | awk '/inet /{print $2}' | cut -d/ -f1)
docker rm -f k3s-m1
docker run -d --privileged --name k3s-m1 \
  --restart unless-stopped \
  --network host \
  -e K3S_CLUSTER_INIT=true \
  -e K3S_KUBECONFIG_MODE=644 \
  -e K3S_TOKEN=$K3S_TOKEN \
  -v /etc/resolv.conf:/etc/resolv.conf:ro \
  rancher/k3s:v1.36.1-k3s1 server \
  --tls-san $TS_IP \
  --advertise-address $TS_IP \
  --node-ip $TS_IP

# Get token and export for use later
export K3S_TOKEN=$(docker exec k3s-m1 cat /var/lib/rancher/k3s/server/node-token)
# Should see:
# K10...::server:...

docker logs -f k3s-m1

# Verify
# See later section
```

### Phase 2: Join VPS 2 as Master

```bash
# Paste token and VPS 1's Tailscale IP from Phase 1
export K3S_TOKEN=<token>
export VPS1_TS_IP=<vps1-tailscale-ip>
TS_IP=$(ip addr show tailscale0 | awk '/inet /{print $2}' | cut -d/ -f1)

docker rm -f k3s-m2
docker run -d --privileged --name k3s-m2 \
  --restart unless-stopped \
  --network host \
  -e K3S_TOKEN=$K3S_TOKEN \
  -e K3S_KUBECONFIG_MODE=644 \
  rancher/k3s:v1.36.1-k3s1 server \
  --server https://$VPS1_TS_IP:6443 \
  --advertise-address $TS_IP \
  --node-ip $TS_IP

docker logs -f k3s-m2
```

### Phase 3: Join VPS 3 as Master

```bash
# Paste token and VPS 1's Tailscale IP from Phase 1
export K3S_TOKEN=<token>
export VPS1_TS_IP=<vps1-tailscale-ip>
TS_IP=$(ip addr show tailscale0 | awk '/inet /{print $2}' | cut -d/ -f1)

docker run -d --privileged --name k3s-m3 \
  --restart unless-stopped \
  --network host \
  -e K3S_TOKEN=$K3S_TOKEN \
  -e K3S_KUBECONFIG_MODE=644 \
  -v /etc/resolv.conf:/etc/resolv.conf:ro \
  rancher/k3s:v1.36.1-k3s1 server \
  --server https://$VPS1_TS_IP:6443 \
  --advertise-address $TS_IP \
  --node-ip $TS_IP

docker logs -f k3s-m3
```

### Verify

```bash
# From VPS 1
# kubectl cli is available with the container
docker exec k3s-m1 kubectl get nodes -o wide

# Should see:
# NAME              STATUS   ROLES                AGE     VERSION        INTERNAL-IP      EXTERNAL-IP   OS-IMAGE           KERNEL-VERSION                 CONTAINER-RUNTIME
# k3s-m1   Ready    control-plane,etcd   7m14s   v1.36.1+k3s1   100.127.166.18   <none>        K3s v1.36.1+k3s1   6.12.88+deb13-amd64 (amd64)    containerd://2.2.3-k3s1
# k3s-m2    Ready    control-plane,etcd   5m58s   v1.36.1+k3s1   100.114.228.3    <none>        K3s v1.36.1+k3s1   6.12.93+rpt-rpi-2712 (arm64)   containerd://2.2.3-k3s1
# k3s-m3   Ready    control-plane,etcd   8m28s   v1.36.1+k3s1   100.91.251.22    <none>        K3s v1.36.1+k3s1   6.12.85+deb13-amd64 (amd64)    containerd://2.2.3-k3s1

# Check pods
docker exec k3s-m1 kubectl get pods -A

# Should see:
# NAMESPACE     NAME                                      READY   STATUS              RESTARTS   AGE
# kube-system   coredns-6648f7576f-rlm57                  0/1     ContainerCreating   0          6m25s
# kube-system   helm-install-traefik-6wm2t                0/1     ContainerCreating   0          6m25s
# kube-system   helm-install-traefik-crd-cb4rz            0/1     ContainerCreating   0          6m25s
# kube-system   local-path-provisioner-58d557dc48-6r6jn   0/1     ContainerCreating   0          6m25s
# kube-system   metrics-server-7c86f97b8d-4hc5d           0/1     ContainerCreating   0          6m25s
```

> **Note**: Replace `<vps1-tailscale-ip>` with VPS 1's Tailscale IP (e.g. `100.91.251.22`). Each node auto-detects its own Tailscale IP via `ip addr show tailscale0`.

## FAQs

<details>
<summary><b>Why Tailscale instead of public/private IPs?</b></summary>

- Private VPC IPs (e.g. `172.17.x.x`) are not routable across different cloud providers or VPCs.
- Public IPs are NATted by the cloud provider — etcd can't bind to them (`bind: cannot assign requested address`).
- Tailscale gives each node a directly bindable, routable `100.x.x.x` IP that works across any network.
</details>

<details>
<summary><b>Why does the agent container fail with port 6444 in use?</b></summary>

When running both server and agent on the same host with `--network host`, the agent's client-side load balancer tries to bind `127.0.0.1:6444`, conflicting with the server. Fix: set `-e K3S_LB_SERVER_PORT=6445` or skip the agent container — K3s server already includes a built-in worker.

</details>

<details>
<summary><b>Why "duplicate hostname" when running agent on the same host as server?</b></summary>

Both containers share the same hostname. Add `--with-node-id` to the agent command to append a random suffix to the node name.

</details>

<details>
<summary><b>Why does containerd fail to pull images from Docker Hub?</b></summary>

The container inherits the host's `/etc/resolv.conf` at creation time. On Chinese cloud VPS (Aliyun), the default DNS (`100.100.2.x`) can't resolve Docker Hub. Fix: `resolvectl dns eth0 8.8.8.8 223.5.5.5` and bind-mount with `-v /etc/resolv.conf:/etc/resolv.conf:ro`.

</details>

<details>
<summary><b>Why did etcd fail with "no route to host" on private IPs?</b></summary>

etcd auto-detects the node's private IP from the network interface. When nodes are on different subnets/VPCs, they can't reach each other. Fix: set `--node-ip` to the Tailscale IP so etcd uses it for peer URLs instead of the private IP.

</details>

<details>
<summary><b>Why "CA cert validation failed" with TLS?</b></summary>

When joining via public IP, the server's TLS certificate doesn't include that IP in its SAN list. Fix: add `--tls-san <public-or-tailscale-ip>` to the first server's bootstrap command.

</details>

<details>
<summary><b>Why `--advertise-address` wasn't enough for etcd?</b></summary>

`--advertise-address` only affects the API server's advertised address. etcd peer URLs use the node IP from `--node-ip` or auto-detection. Both flags are needed for full cross-node communication.

</details>

<details>
<summary><b>Why are pods stuck in ContainerCreating?</b></summary>

containerd inside the K3s container needs to pull multiple images: `rancher/mirrored-pause:3.6`, `rancher/mirrored-coredns`, `rancher/klipper-helm`, `rancher/mirrored-metrics-server`, `rancher/mirrored-local-path-provisioner`, traefik, etc.

If the container's DNS can't resolve `registry-1.docker.io` (common on Chinese cloud VPS), pre-pull on the host and import:

```bash
# Fix DNS first (run once on host)
resolvectl dns eth0 8.8.8.8 223.5.5.5

# Pre-pull pause image and import into K3s containerd
docker pull rancher/mirrored-pause:3.6
docker save rancher/mirrored-pause:3.6 | docker exec -i <container> ctr -n k8s.io images import -
```

Repeat for other images if they stay stuck. After importing, K3s retries automatically.

**Prevention**: Mount the host's resolv.conf with `-v /etc/resolv.conf:/etc/resolv.conf:ro` when creating the container.

</details>
