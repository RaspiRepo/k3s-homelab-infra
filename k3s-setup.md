# K3s Hybrid Cluster Setup, kube-proxy Migration & Monitoring Runbook

Complete operational runbook for managing a three-node hybrid K3s cluster (`v1.37.2+k3s1`).

This document covers:

- Cluster inventory and hardware labeling
- `kube-proxy` backend architecture (`nftables` and `iptables`)
- Node-specific K3s configuration
- WSL2 IPv6 and `nftables` compatibility
- NVIDIA Jetson Tegra kernel limitations
- Cross-node DNS and Flannel VXLAN troubleshooting
- Prometheus and Grafana deployment
- `kube-proxy` metrics and PromQL queries
- Cluster verification and monitoring access

---

## 1\. Cluster Architecture & Inventory

### 1.1 Node Inventory

| Node | Role | Architecture | GPU / Hardware | OS / Environment | Kernel | IP | Proxy Backend |
| --- | --- | --- | --- | --- | --- | --- | --- |
| `bluebird` | Control Plane | amd64 | RTX 5060 Ti (16GB) | Ubuntu 24.04.5 LTS (WSL2) | `6.6.87.2-microsoft-standard-WSL2` | `172.30.1.175` | `nftables` |
| `ubuntu-xps-8930` | Worker | amd64 | RTX 2060 (8GB) | Ubuntu 26.04.1 LTS (Native) | `6.8.0-146-generic` | `172.30.1.10` | `nftables` |
| `jetson-orin` | Worker | arm64 | Jetson Orin (8GB) | Ubuntu 24.04.4 LTS (Tegra) | `6.8.12-1021-tegra` | `172.30.3.89` | `iptables` |

### 1.2 Final Proxy Architecture

| Node | Backend | Configuration |
| --- | --- | --- |
| `bluebird` | `nftables` | IPv4 single-stack; IPv6 disabled |
| `ubuntu-xps-8930` | `nftables` | Native IPv4/IPv6 support |
| `jetson-orin` | `iptables` | Default K3s backend retained |

**Architecture decision:** Use a hybrid proxy architecture. The two x86 nodes use `nftables`, while the NVIDIA Jetson remains on `iptables` because of its Tegra kernel compatibility limitations.

Different nodes can use different local `kube-proxy` backends while participating in the same Kubernetes cluster.

---

## 2\. Background: kube-proxy Backends

Based on Kubernetes SIG Network guidance and KEP-5495, [Scheduling the removal of the kube-proxy IPVS backend](<https://kubernetes.io/blog/2026/11/16/ipvs-removal/>).

### 2.1 Backend Comparison

| Feature | `iptables` | `ipvs` | `nftables` |
| --- | --- | --- | --- |
| Kubernetes support | Supported | Deprecated | Recommended |
| Kernel requirements | Broad compatibility | IPVS kernel support | Kernel 5.13+ |
| Rule updates | Incremental optimizations available | Limited batch update support | Atomic incremental updates |
| Scaling characteristics | Improved in recent releases | Scheduler/state limitations | Efficient verdict maps |
| Compatibility | Broad | Legacy | Requires appropriate kernel support |

### 2.2 Why nftables?

- Reached General Availability in Kubernetes 1.33 after being introduced in 1.29.
- Uses verdict maps (`vmap`) to improve rule lookup efficiency.
- Supports atomic incremental updates.
- Provides improved performance and scalability for large Service endpoint sets.
- Avoids some limitations associated with the IPVS implementation.

Kubernetes 1.36+ includes additional `nftables` performance and scaling optimizations.

### 2.3 Why IPVS Is Being Removed

IPVS has several architectural limitations:

- Its kernel API lacks efficient batch updates.
- Some Kubernetes Service behaviors require fallback to `iptables` and `ipset`.
- IPVS scheduler state is not shared across nodes.
- Custom schedulers such as Maglev (`mh`) cannot provide their intended cluster-wide benefits through independent node-local state.

Kubernetes deprecated IPVS in 1.35, with removal targeted for 1.40.

### 2.4 Why iptables Remains a Valid Fallback

Modern `iptables` mode includes performance improvements:

- Reduced redundant rules.
- Lower resynchronization memory usage.
- Optimistic state checks.
- Incremental updates through KEP-3453.

For hardware with incomplete `nftables` kernel support, `iptables` remains a practical fallback.

---


## 3. Adding a New Worker Node to the K3s Cluster
This section describes how to join an additional Linux machine to the existing K3s cluster using the K3s server token.


### 4.2 Step 1: Retrieve the Worker Join Token

Run on the control-plane node (`bluebird`):

bash

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```

Example output:

text

```text
K10xxxxxxxxxxxxxxxxxxxxxxxx::server:xxxxxxxxxxxxxxxx
```

Copy the complete token securely.

Replace `new-worker` with the desired hostname.

### 4.4 Step 3: Install and Join the Worker

Run on the new worker machine.

Set the token obtained from the control plane:

bash

```bash
export K3S_URL="https://172.30.1.175:6443"
export K3S_TOKEN="PASTE_YOUR_NODE_TOKEN_HERE"
curl -sfL https://get.k3s.io | K3S_URL="$K3S_URL" K3S_TOKEN="$K3S_TOKEN" sh -

```


## 3.1\. Node Labeling

Apply hardware, GPU, and role labels to all three nodes.

### 3.2 Control Plane: bluebird

Hardware: RTX 5060 Ti (16GB)

bash

```bash
kubectl label node bluebird \
  gpu=true \
  gpu-type=rtx5060ti \
  gpu-memory=16GB \
  node-type=desktop \
  --overwrite
```

### 3.3 Worker 1: ubuntu-xps-8930

Hardware: RTX 2060 (8GB)

bash

```bash
kubectl label node ubuntu-xps-8930 \
  node-role.kubernetes.io/worker=worker \
  gpu=true \
  gpu-type=rtx2060 \
  gpu-memory=8GB \
  node-type=desktop \
  --overwrite
```

### 3.4 Worker 2: jetson-orin

Hardware: NVIDIA Jetson Orin (8GB, ARM64)

bash

```bash
kubectl label node jetson-orin \
  node-role.kubernetes.io/worker=worker \
  gpu=true \
  gpu-type=orin \
  gpu-memory=8GB \
  node-type=edge \
  --overwrite
```

### 3.5 Verify Labels

bash

```bash
kubectl get nodes --show-labels
```
---

## 4\. kube-proxy Migration & Configuration

### 4.1 Prerequisites

| Component | Requirement |
| --- | --- |
| K3s | `v1.37.2+k3s1` |
| Kubernetes | 1.36+ recommended for newer `nftables` optimizations |
| Linux kernel | 5.13+ for `nftables` proxy mode |
| nftables CLI | 1.0.1+ |
| Metrics port | TCP 10249 |

Install the `nftables` CLI if necessary:

bash

```bash
sudo apt update
sudo apt install -y nftables
```

Check installed versions:

bash

```bash
uname -r
nft --version
```

### 4.2 Control Plane: bluebird (WSL2)

#### Step 1: Check WSL2 Environment

bash

```bash
uname -r
cat /etc/wsl.conf
nft --version
```

Expected kernel:

text

```text
6.6.87.2-microsoft-standard-WSL2
```

Expected WSL configuration:

ini

```ini
[boot]
systemd=true
```

#### Step 2: Test IPv4 nftables Support

Verify that the kernel supports IPv4 `nftables` NAT tables:

bash

```bash
sudo nft add table ip k3s_test && \
sudo nft 'add chain ip k3s_test postrouting { type nat hook postrouting priority 100 ; }' && \
sudo nft delete table ip k3s_test
```

#### Step 3: Disable IPv6

The standard WSL2 kernel used by this node lacks the required `CONFIG_NFT_FIB_IPV6` support.

This can cause IPv6 proxy rule synchronization failures:

text

```text
Error: Could not process rule: No such file or directory
```

Disable IPv6 to operate the node in IPv4 single-stack mode:

bash

```bash
echo -e "net.ipv6.conf.all.disable_ipv6 = 1\nnet.ipv6.conf.default.disable_ipv6 = 1" \
  | sudo tee /etc/sysctl.d/99-disable-ipv6.conf

sudo sysctl --system
```

#### Step 4: Configure kube-proxy

Create the K3s configuration:

bash

```bash
sudo mkdir -p /etc/rancher/k3s

cat <<EOF | sudo tee /etc/rancher/k3s/config.yaml
kube-proxy-arg:
  - "proxy-mode=nftables"
  - "metrics-bind-address=0.0.0.0:10249"
EOF
```

Restart the control-plane service:

bash

```bash
sudo systemctl restart k3s
```

#### Step 5: Verify kube-proxy

bash

```bash
sudo journalctl -u k3s -n 25 --no-pager
```

Expected log messages include:

text

```text
Kernel support disabled for family ipFamily="IPv6"
kube-proxy running in single-stack mode ipFamily="IPv4"
Using nftables Proxier
```

Verify the generated rules:

bash

```bash
sudo nft list table ip kube-proxy | head -n 20
```

---

### 4.3 Worker 1: ubuntu-xps-8930 (Native Ubuntu)

Native Ubuntu provides the required IPv4 and IPv6 `nftables` kernel support for this configuration.

IPv6 does not need to be disabled.

#### Step 1: Configure kube-proxy

bash

```bash
sudo mkdir -p /etc/rancher/k3s

cat <<EOF | sudo tee /etc/rancher/k3s/config.yaml
kube-proxy-arg:
  - "proxy-mode=nftables"
  - "metrics-bind-address=0.0.0.0:10249"
EOF
```

#### Step 2: Restart K3s Agent

bash

```bash
sudo systemctl restart k3s-agent
```

#### Step 3: Verify Configuration

bash

```bash
sudo journalctl -u k3s-agent -n 25 --no-pager
```

Inspect the generated rules:

bash

```bash
sudo nft list table ip kube-proxy
```

---

### 4.4 Worker 2: jetson-orin (NVIDIA Jetson / Tegra)

#### Kernel Limitation

The NVIDIA Tegra kernel (`6.8.12-1021-tegra`) has incomplete `nftables` kernel support in the current configuration.

Missing or unavailable functionality has included:

- `CONFIG_NFT_NUMGEN`
- `CONFIG_NFT_FIB_IPV6`

This prevents the node from reliably creating the required `kube-proxy` rules.

A reported failure includes:

text

```text
dial tcp 10.43.0.1:443: connect: no route to host
```

**Decision:** Keep `jetson-orin` on the default `iptables` backend.

This is compatible with the mixed-backend cluster architecture.

#### Step 1: Configure Metrics Only

Do not explicitly set `proxy-mode=nftables` on this node.

bash

```bash
sudo mkdir -p /etc/rancher/k3s

cat <<EOF | sudo tee /etc/rancher/k3s/config.yaml
kube-proxy-arg:
  - "metrics-bind-address=0.0.0.0:10249"
EOF
```

#### Step 2: Restart K3s Agent

bash

```bash
sudo systemctl restart k3s-agent
```

#### Step 3: Verify Proxy Status

bash

```bash
sudo journalctl -u k3s-agent -n 25 --no-pager \
  | grep -iE "proxy|nft|error|fail"
```

**Important:** Do not run `sudo nft flush ruleset` as a routine migration step. It removes unrelated firewall rules and can disrupt networking. Only perform firewall cleanup as part of a deliberate recovery procedure after inspecting the existing ruleset.

---

## 5\. Cross-Node DNS & Flannel VXLAN Troubleshooting

### 5.1 Problem Description

When CoreDNS runs on the WSL2 control-plane node (`bluebird`), pods on native worker nodes may experience DNS timeouts.

Example:

text

```text
lookup kube-prometheus-stack-prometheus.monitoring: i/o timeout
```

Potential causes include:

- Windows Firewall blocking inbound Flannel VXLAN traffic.
- UDP 8472 connectivity problems.
- VXLAN interface checksum offloading issues.
- Cross-node networking problems between WSL2 and native Linux.

### 5.2 Step 1: Move CoreDNS to the Native Worker

As an immediate workaround, pin CoreDNS to `ubuntu-xps-8930`.

bash

```bash
kubectl patch deployment coredns -n kube-system \
  -p '{"spec":{"template":{"spec":{"nodeSelector":{"kubernetes.io/hostname":"ubuntu-xps-8930"}}}}}'
```

Verify CoreDNS placement:

bash

```bash
kubectl get pods -n kube-system -o wide | grep coredns
```

This avoids sending DNS requests across the WSL2 VXLAN boundary.

**Note:** This patch is a direct workload configuration change. For persistent Helm-managed configuration, use the appropriate K3s CoreDNS HelmChartConfig or chart values rather than relying exclusively on a manual patch.

### 5.3 Step 2: Allow Flannel VXLAN Through Windows Firewall

Run the following in Windows PowerShell as Administrator on the host running `bluebird`:

powershell

```powershell
New-NetFirewallRule `
  -DisplayName "K3s Flannel VXLAN" `
  -Direction Inbound `
  -Protocol UDP `
  -LocalPort 8472 `
  -Action Allow
```

Flannel VXLAN uses UDP port 8472 by default in this configuration.

### 5.4 Step 3: Disable VXLAN TX Checksum Offloading

If cross-node pod traffic or ping continues to drop, test disabling TX checksum offloading.

Run on the affected cluster nodes:

bash

```bash
sudo ethtool -K flannel.1 tx-checksum-ip-generic off
```

Verify the interface:

bash

```bash
ip link show flannel.1
ethtool -k flannel.1
```

---

## 6\. Monitoring Stack: kube-prometheus-stack

Deploy Prometheus, Grafana, Alertmanager, node-exporter, and kube-state-metrics using the Prometheus Community Helm chart.

Monitoring components are pinned to `ubuntu-xps-8930` to avoid WSL2 networking and scheduling complications.

### 6.1 Monitoring Architecture

| Component | Configuration |
| --- | --- |
| Prometheus | Runs on `ubuntu-xps-8930` |
| Grafana | Runs on `ubuntu-xps-8930` |
| Prometheus Operator | Runs on `ubuntu-xps-8930` |
| Alertmanager | Runs on `ubuntu-xps-8930` |
| kube-state-metrics | Runs on `ubuntu-xps-8930` |
| node-exporter | Enabled |
| kube-proxy metrics | Static targets on all three nodes |
| Metrics port | TCP 10249 |

### 6.2 Create monitoring-values.yaml

Create the Helm values file:

bash

```bash
nano monitoring-values.yaml
```

Configuration:

yaml

```yaml
defaultRules:
  create: true

prometheusOperator:
  enabled: true
  nodeSelector:
    kubernetes.io/hostname: ubuntu-xps-8930
  admissionWebhooks:
    patch:
      nodeSelector:
        kubernetes.io/hostname: ubuntu-xps-8930

grafana:
  enabled: true
  nodeSelector:
    kubernetes.io/hostname: ubuntu-xps-8930
  adminUser: admin
  adminPassword: "CHANGE_ME_TO_A_SECURE_PASSWORD"
  grafana.ini:
    analytics:
      reporting_enabled: false
    server:
      domain: grafana.rtx5060.lan
      root_url: http://rtx5060.lan

prometheus:
  enabled: true
  prometheusSpec:
    nodeSelector:
      kubernetes.io/hostname: ubuntu-xps-8930
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    retention: 15d
    resources:
      requests:
        cpu: 500m
        memory: 1Gi

alertmanager:
  enabled: true
  alertmanagerSpec:
    nodeSelector:
      kubernetes.io/hostname: ubuntu-xps-8930

kubeStateMetrics:
  enabled: true

kube-state-metrics:
  nodeSelector:
    kubernetes.io/hostname: ubuntu-xps-8930

nodeExporter:
  enabled: true

kubeProxy:
  enabled: true
  endpoints:
    - 172.30.1.175
    - 172.30.1.10
    - 172.30.3.89
  service:
    enabled: true
    port: 10249
    targetPort: 10249
```

**Security:** Replace the Grafana password placeholder before deployment. For production, prefer Kubernetes Secrets or an external secret-management solution rather than storing passwords directly in a values file.

### 6.3 Configure Helm Repository

bash

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

helm repo update
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
helm upgrade --install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f monitoring-values.yaml

kubectl get pods -n monitoring -o wide
kubectl get svc -n monitoring
```

**Important configuration note:** Verify the installed chart version's `kubeProxy` values schema before relying on the static `endpoints` configuration above. The chart's built-in kube-proxy monitoring configuration can vary between releases. Also inspect and remove conflicting K3s placeholder Service or Endpoints objects only when their ownership and purpose have been confirmed. Do not blindly delete monitoring resources.

### 6.5 Verify kube-proxy Metrics

The three nodes expose metrics on port 10249.

| Node | IP | Backend | Metrics Endpoint |
| --- | --- | --- | --- |
| `bluebird` | `172.30.1.175` | `nftables` | `http://172.30.1.175:10249/metrics` |
| `ubuntu-xps-8930` | `172.30.1.10` | `nftables` | `http://172.30.1.10:10249/metrics` |
| `jetson-orin` | `172.30.3.89` | `iptables` | `http://172.30.3.89:10249/metrics` |

Test locally on each node:

```bash
curl -s http://127.0.0.1:10249/metrics | head
curl -s http://172.30.1.175:10249/metrics | head
curl -s http://172.30.1.10:10249/metrics | head
curl -s http://172.30.3.89:10249/metrics | head
```

Ensure firewall rules permit TCP 10249 from the monitoring node.

---

## 7\. Prometheus & Grafana Queries

Use Prometheus or Grafana Explore to compare `nftables` and `iptables` proxy performance.

### 7.1 Average Proxy Rule Synchronization Duration

Average synchronization duration over five minutes, in milliseconds:

promql

```promql
rate(kubeproxy_sync_proxy_rules_duration_seconds_sum[5m])
/
rate(kubeproxy_sync_proxy_rules_duration_seconds_count[5m])
* 1000
```

### 7.2 P99 Proxy Rule Synchronization Duration

promql

```promql
histogram_quantile(
  0.99,
  sum(
    rate(kubeproxy_sync_proxy_rules_duration_seconds_bucket[5m])
  ) by (le, instance)
) * 1000
```
---

## 8\. Standalone Grafana Deployment

Use this option only if deploying Grafana independently rather than using the Grafana instance included in `kube-prometheus-stack`.

### 8.1 Create grafana-values.yaml

yaml

```yaml
adminUser: admin
adminPassword: "CHANGE_ME_TO_A_SECURE_PASSWORD"

nodeSelector:
  kubernetes.io/arch: amd64
  node-type: desktop

service:
  type: NodePort
  nodePort: 30000

persistence:
  enabled: true
  size: 5Gi
```

### 8.2 Add Grafana Helm Repository

bash

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

---

## 9\. Cluster-Wide Verification

Run these commands after completing the configuration and monitoring deployment.

### 9.1 Verify Cluster Nodes

bash

```bash
kubectl get nodes -o wide
kubectl get nodes --show-labels
kubectl get svc,endpoints,endpointslices -n kube-system \
  | grep kube-proxy
kubectl get pods -A -o wide
```


### 9.8 Retrieve Grafana Admin Password

If using the chart-generated admin Secret:

bash

```bash
kubectl get secret \
  --namespace monitoring \
  -l app.kubernetes.io/component=admin-secret \
  -o jsonpath="{.items[0].data.admin-password}" \
  | base64 --decode

echo
```

### 9.9 Port-Forward Grafana

For `kube-prometheus-stack`:

bash

```bash
kubectl port-forward \
  -n monitoring \
  svc/kube-prometheus-stack-grafana \
  3000:80
```

For standalone Grafana:

bash

```bash
kubectl port-forward \
  -n monitoring \
  svc/grafana \
  3000:80
```

Access Grafana:

text

```text
http://localhost:3000
```

---

## 10\. Final Architecture Summary

| Component | bluebird | ubuntu-xps-8930 | jetson-orin |
| --- | --- | --- | --- |
| Role | Control Plane | Worker | Worker |
| Architecture | amd64 | amd64 | arm64 |
| Hardware | RTX 5060 Ti | RTX 2060 | Jetson Orin |
| Environment | WSL2 | Native Ubuntu | NVIDIA Tegra |
| kube-proxy | nftables | nftables | iptables |
| IPv6 | Disabled | Enabled | Default configuration |
| Metrics | 10249 | 10249 | 10249 |
| GPU Label | rtx5060ti | rtx2060 | orin |
| Node Type | desktop | desktop | edge |
