# 🛠️ Kubernetes Dashboard Installation Guide

[**English**](README.en.md) | [**한국어**](README.md)

This guide explains how to install and configure Kubernetes Dashboard using Helm.

## 📋 Overview

Kubernetes Dashboard is a web-based UI that allows you to deploy containerized applications, troubleshoot issues, and manage cluster resources.

> **What is Helm?**
> Helm is the package manager for Kubernetes (similar to `apt` for Linux or `pip` for Python).

---

## 🚀 Installation Steps

### 1. Add Helm Repository

```bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
```

### 2. Prepare Configuration (values.yaml)

We modify `values.yaml` to deploy the Dashboard only on the Master Node.

#### 2.1 Download Default Values
```bash
helm show values kubernetes-dashboard/kubernetes-dashboard > values.yaml
```

#### 2.2 Modify values.yaml
Add `nodeSelector` and `tolerations` to schedule Pods on the Control Plane (Master Node).

```yaml
nodeSelector:
  kubernetes.io/hostname: "<MASTER_NODE_NAME>" # Check with: kubectl get nodes
tolerations:
- key: "node-role.kubernetes.io/control-plane"
  operator: "Exists"
  effect: "NoSchedule"
```

### 3. Install via Helm

Create the `kubernetes-dashboard` namespace and install.

```bash
helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard \
  -f values.yaml \
  --version 7.5.0 \
  --namespace kubernetes-dashboard \
  --create-namespace
```

Verify installation:
```bash
kubectl get svc -n kubernetes-dashboard
```

---

## 🌐 External Access (NodePort)

Configure `NodePort` to access the Dashboard from an external browser.

1. Edit the service:
    ```bash
    kubectl edit service kubernetes-dashboard-kong-proxy -n kubernetes-dashboard
    ```
2. Set `type: NodePort` and `nodePort: 31000` under `spec.ports`:

    ```yaml
    spec:
      ports:
      - name: kong-proxy-tls
        nodePort: 31000  # External Access Port
        port: 443
        protocol: TCP
        targetPort: 8443
      type: NodePort     # Change ClusterIP to NodePort
    ```

You can now access the dashboard at `https://<Master_IP>:31000`.

---

## 🔑 Admin User Configuration

Create an admin user with full access to the cluster.

### 1. Create ServiceAccount & RoleBinding

Create a file named `dashboard-admin.yaml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

Apply the configuration:
```bash
kubectl apply -f dashboard-admin.yaml
```

### 2. Generate Login Token
Generate a token to log in (use `--duration` to extend validity).

```bash
kubectl -n kubernetes-dashboard create token admin-user --duration 720h
```

---

## 📊 Install Metrics Server

**Metrics Server** is required to view CPU and Memory usage in the Dashboard.

### 1. Configuration (components.yaml)
The provided [components.yaml](components.yaml) file includes these changes:
- **Namespace**: Consolidated into `kubernetes-dashboard`
- **NodeSelector**: Deployed on Master Node
- **Kubelet Communication**: Access allowed to `kube-system` metadata

### 2. Install & Verify

```bash
kubectl apply -f components.yaml
```

Verify by running `kubectl top nodes`.

---

### 💡 Tips

- **Delete Namespace**: To completely remove the installation.
  ```bash
  kubectl delete ns kubernetes-dashboard
  ```
- **Force Delete**: If deletion hangs, remove the finalizer.
  ```bash
  kubectl patch namespace kubernetes-dashboard -p '{"metadata":{"finalizers":[]}}' --type=merge
  ```
