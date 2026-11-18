# NVIDIA DCGM Exporter Deployment

This guide covers deploying the NVIDIA `dcgm-exporter` to the `monitoring` namespace using Helm to collect GPU telemetry metrics.

## Deployment Command

Run the following command to install the chart with custom values:

```bash
helm install dcgm-exporter gpu-helm-charts/dcgm-exporter \
  -n monitoring \
  -f dcgm-exporter/dcgm-values.yaml
```

## Configuration (`dcgm-values.yaml`)

```yaml
service:
  enabled: true
  type: ClusterIP
  port: 9400

serviceMonitor:
  enabled: true
  interval: 15s
  scrapeTimeout: 10s

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 200m
    memory: 512Mi

tolerations:
  - operator: Exists

# Injects the NVIDIA runtime so the container can access /dev/nvidia paths
runtimeClassName: nvidia

# Ensures the container runs with host network mapping
hostNetwork: true

# Configures the pod to run with sufficient permissions
securityContext:
  allowPrivilegeEscalation: true
  readOnlyRootFilesystem: false
  runAsNonRoot: false
  runAsUser: 0
  capabilities:
    add:
      - SYS_ADMIN

# Restricts deployment to GPU nodes only
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: NotIn
          values:
          - "agentx"
```

## Key Configuration Details

- **Prometheus Integration:** Enables the `ClusterIP` service on port `9400` and configures a `ServiceMonitor` to scrape GPU metrics every `15s`.
- **GPU Hardware Access:** Uses `runtimeClassName: nvidia`, `hostNetwork: true`, and `SYS_ADMIN` privileges to read low-level hardware telemetry from `/dev/nvidia`.
- **Node Scheduling:** Applies an `Exists` toleration to run on tainted GPU nodes while using `nodeAffinity` (`NotIn`) to exclude the non-GPU `agentx` control/worker node.
