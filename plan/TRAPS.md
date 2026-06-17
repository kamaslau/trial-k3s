# Traps Encountered

Lessons learned while trying to set up a cross-platform HA K3s cluster with Docker.

---

## 1. k3d etcd Bridge IP — Cross-Host Invisible

**Symptom**: Other hosts can't join as masters — `MemberAdd request timed out`.

**Cause**: k3d runs containers on a Docker bridge network (`172.18.0.0/16`). etcd peers advertise their bridge IP (`172.18.0.3:2380`), which is unreachable from other physical hosts.

```text
Adding member mba=192.168.31.206:2380 to etcd cluster [k3d-server=https://172.18.0.3:2380]
                                                       ^^^^^^^^^^ unreachable from LAN
```

**Fix**: Don't use k3d for cross-host clusters. Use plain `docker run rancher/k3s` with `--network host` or port mapping instead.

---

## 2. Docker Desktop WSL2 — `--network host` Doesn't Expose to LAN

**Symptom**: Container starts, but port 6443 isn't listening on the Windows host IP (`192.168.31.131`). `netstat -an | findstr :6443` returns nothing.

**Cause**: Docker Desktop on Windows runs containers inside a Hyper-V WSL2 VM. `--network host` shares the VM's network namespace, not the Windows host's. The port is only reachable from within WSL2.

```text
host (192.168.31.131)  ──╯  WSL2 VM (172.x.x.x)  ──✓  container (--network host)
```

**Fix**: Use `-p 6443:6443 -p 2380:2380` (port mapping) instead of `--network host` on WSL2.

---

## 3. Docker Desktop macOS — `--network host` Can't Bind LAN IPs

**Symptom**: etcd fails to bind to the advertised LAN IP:

```text
listen tcp 192.168.31.206:2380: bind: cannot assign requested address
```

**Cause**: Same fundamental issue as WSL2 — Docker Desktop on macOS runs containers in a Linux VM. `--network host` shares the VM's network stack, which has different IPs than the macOS host. The container can't bind to the Mac's WiFi IP.

**Fix**: Use port mapping on macOS too, or use Tailscale to create a virtual interface the container CAN bind to.

---

## 4. Pi Dual-Homed — TLS Cert Uses Wired IP, Not WiFi

**Symptom**: MBA/PC i7 fail to validate bootstrap token:

```text
x509: certificate is valid for 10.43.0.1, 127.0.0.1, 192.168.1.3, ..., not 192.168.31.129
```

**Cause**: Pi has both `eth0` (wired, `192.168.1.3`) and `wlan0` (WiFi, `192.168.31.129`). K3s auto-detects the default route interface (`eth0`) and generates TLS certificates for that IP. Other hosts connect via WiFi IP which isn't in the cert.

**Fix**: Always pass `--tls-san <wifi-ip>` on the bootstrap node when the cluster communicates over a different interface than the default route.

---

## 5. MBA M2 — IPv6 Auto-Detection Conflicts with IPv4 CIDR

**Symptom**: K3s server fails to start:

```text
Error: cluster-cidr: [10.42.0.0/16] and node-ip: [fdc4:f303:9324::3], must share the same IP version
```

**Cause**: macOS has IPv6 enabled. K3s auto-detects both IPv4 and IPv6 addresses. Without explicit `--node-ip`, it picks the IPv6 address, which conflicts with the IPv4-only cluster CIDR.

**Fix**: Always set `--node-ip <ipv4-lan-ip>` when joining from macOS hosts.

---

## 6. Docker Desktop — Flannel Can't Find Default Route

**Symptom**: K3s starts, flannel crashes:

```text
Flannel exited: failed to find the interface: failed to get default interface: unable to find default route
```

**Cause**: Docker Desktop's `--network host` mode doesn't provide a standard network route to the container. Flannel needs a physical interface to bind to for VXLAN.

**Fix**: Use `--flannel-iface <interface>` to specify the interface explicitly, or avoid `--network host` on Docker Desktop entirely.

---

## 7. etcd — `--initial-cluster` vs `--advertise-address` Mismatch

**Symptom**: Bootstrap fails:

```text
--initial-cluster has node=https://172.17.0.3:2380 but missing from --initial-advertise-peer-urls=https://192.168.31.131:2380
```

**Cause**: `--advertise-address` only affects kube-apiserver's advertisement. etcd's `initial-cluster` URL is built from the container's detected IP (Docker bridge). Setting `--etcd-arg "--initial-advertise-peer-urls=..."` changes the advertise URLs but K3s still builds `initial-cluster` from the container IP. They must match, but can't easily be overridden.

**Fix**: Ensure the container's detected IP matches what you want etcd to advertise. On Linux, `--network host` gives the correct host IP. On Docker Desktop, use Tailscale or macvlan to give the container a routable IP.

---

## 8. K3s Config Mismatch — Joining Server vs Bootstrap Defaults

**Symptom**: Joining server fails:

```text
critical configuration mismatched: ClusterDNSs, ClusterIPRanges
```

**Cause**: When bootstrap is created by k3d (which may set different defaults) and joining server uses `docker run rancher/k3s`, the two can have different default CIDR/DNS settings. K3s validates all servers agree.

**Fix**: Ensure all servers use the same version of K3s and pass the same `--cluster-cidr`, `--service-cidr`, and `--cluster-dns` flags if defaults might differ.

---

## 9. Cross-Platform Container Networking Summary

| Platform             |           `--network host`           | Port mapping (`-p`) |  macvlan   | Tailscale |
| -------------------- | :----------------------------------: | :-----------------: | :--------: | :-------: |
| Linux (Pi)           |               ✅ Works               |      ✅ Works       |  ✅ Works  | ✅ Works  |
| macOS Docker Desktop | ⚠️ Flannel fails, can't bind LAN IPs |      ✅ Works       | ⚠️ Limited | ✅ Works  |
| Windows WSL2         |       ❌ Doesn't expose to LAN       |      ✅ Works       | ⚠️ Complex | ✅ Works  |
| Aliyun (Debian)      |               ✅ Works               |      ✅ Works       |  ✅ Works  | ✅ Works  |

**Tailscale is the only solution that works consistently across all platforms** for `--network host` because it creates a real virtual network interface that containers can bind to.
