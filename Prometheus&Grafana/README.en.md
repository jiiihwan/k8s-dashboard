# ⚙️ Prometheus & Grafana Installation Guide

[**English**](README.en.md) | [**한국어**](README.md)

This guide explains how to install and configure Prometheus and Grafana using Helm.
The main components of the monitoring stack (Alertmanager, Grafana, Prometheus, etc.) will be installed on the **Master Node**.

## 🚀 Installation Steps

### 1. Install Helm & Add Repos

Install Helm if it is not already installed.
```bash
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

Add Helm repositories and update.
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### 2. Create Namespace & Label Node

Create a dedicated namespace for monitoring.
```bash
kubectl create namespace monitoring
```

Label the Master Node with `key=master` to ensure monitoring pods are scheduled there.
```bash
kubectl label nodes <MASTER_NODE_NAME> key=master
# Verify: kubectl get nodes --show-labels
```

### 3. Install Prometheus Stack

Install `kube-prometheus-stack`. The command below uses `nodeSelector` to deploy key components to the Master Node.

> **Note**: It is recommended to use `values.yaml` for more detailed configurations. See the [Helm Configuration Guide](helm_setting.en.md).

```bash
helm upgrade --install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring \
  --set prometheus.prometheusSpec.nodeSelector.key=master \
  --set prometheus.prometheusSpec.tolerations[0].key="node-role.kubernetes.io/control-plane" \
  --set prometheus.prometheusSpec.tolerations[0].operator="Exists" \
  --set prometheus.prometheusSpec.tolerations[0].effect="NoSchedule" \
  --set alertmanager.alertmanagerSpec.nodeSelector.key=master \
  --set alertmanager.alertmanagerSpec.tolerations[0].key="node-role.kubernetes.io/control-plane" \
  --set alertmanager.alertmanagerSpec.tolerations[0].operator="Exists" \
  --set alertmanager.alertmanagerSpec.tolerations[0].effect="NoSchedule" \
  --set grafana.nodeSelector.key=master \
  --set grafana.tolerations[0].key="node-role.kubernetes.io/control-plane" \
  --set grafana.tolerations[0].operator="Exists" \
  --set grafana.tolerations[0].effect="NoSchedule" \
  --set prometheusOperator.nodeSelector.key=master \
  --set prometheusOperator.tolerations[0].key="node-role.kubernetes.io/control-plane" \
  --set prometheusOperator.tolerations[0].operator="Exists" \
  --set prometheusOperator.tolerations[0].effect="NoSchedule" \
  --set kube-state-metrics.nodeSelector.key=master \
  --set kube-state-metrics.tolerations[0].key="node-role.kubernetes.io/control-plane" \
  --set kube-state-metrics.tolerations[0].operator="Exists" \
  --set kube-state-metrics.tolerations[0].effect="NoSchedule"
```

Verify installation:
```bash
kubectl get svc -n monitoring
```

---

## 🌐 External Access (NodePort)

Change the Service type to NodePort to access Prometheus and Grafana externally.
(Example ports: Prometheus `31001`, Grafana `31002`)

### 1. Edit Prometheus Service
```bash
kubectl edit svc prometheus-kube-prometheus-prometheus -n monitoring
```
```yaml
spec:
  type: NodePort
  ports:
    - name: http-web
      port: 9090
      nodePort: 31001  # Specify Port
```

### 2. Edit Grafana Service
```bash
kubectl edit svc prometheus-grafana -n monitoring
```
```yaml
spec:
  type: NodePort
  ports:
    - name: http-web
      port: 80
      nodePort: 31002  # Specify Port
```

### 3. Port Forwarding (If using Linux/IPTables)
Open the ports using iptables if necessary.
```bash
sudo iptables -I INPUT -p tcp --dport 31001 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 31002 -j ACCEPT
```

---

## 📚 Additional Guides

- **[Helm Configuration Guide](helm_setting.md)**: Advanced configuration using `values.yaml`
- **[Troubleshooting Guide](problem_solving.en.md)**: Node Exporter issues on Jetson Orin Nano
