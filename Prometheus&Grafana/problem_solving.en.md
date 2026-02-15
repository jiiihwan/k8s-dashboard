# 🔧 Troubleshooting: Node Exporter Collection Issue

[**English**](problem_solving.en.md) | [**한국어**](problem_solving.md)

## 🔴 Issue

**Symptom**:
- Node Exporter on the Master Node works fine, but metrics from **Jetson Orin Nano (Worker Node)** are not collected by Prometheus.
- Logs show errors related to CPU Frequency collection or file size warnings.

**Cause**:
- Suspected compatibility issue with the `thermal_zone` collector on Jetson Orin Nano. (Related Issue: [GitHub Issue #3071](https://github.com/prometheus/node_exporter/issues/3071))

---

## 🟢 Solution

Disable the `thermal_zone` collector by adding the `--no-collector.thermal_zone` flag to the Node Exporter arguments.

### Edit DaemonSet

1. Edit the Node Exporter DaemonSet.
    ```bash
    kubectl edit daemonset prometheus-prometheus-node-exporter -n monitoring
    ```

2. Add the flag to `args` under the `containers` section.

    ```yaml
    spec:
      containers:
      - name: node-exporter
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/host/root
        - --no-collector.thermal_zone  # <-- Add this line
    ```

3. Save and exit. The DaemonSet will restart the pods, and metrics collection should resume normally.
