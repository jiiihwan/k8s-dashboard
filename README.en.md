# 💻 k8s-dashboard

[**English**](README.en.md) | [**한국어**](README.md)

> ### k8s dashboard with real-time resource utilization

A custom dashboard implemented using **Prometheus** and **Grafana** to monitor real-time resource usage of nodes, including GPU utilization, which is not supported by the default Kubernetes dashboard.

## 🌟 Features & Supported Resources

Real-time monitoring for the following resources:

1.  **CPU** utilization (%)
    - Per-core utilization
2.  **GPU** utilization (%)
3.  **RAM** utilization (GB & %)
4.  **Network** utilization (Bit/sec)
    - Transmitted & Received
5.  **NPU** utilization (%) 

## 🧱 System Architecture

This system operates on a Kubernetes cluster, where Prometheus collects metrics from exporters installed on each worker node and visualizes them in real-time via Grafana.

![System Architecture](https://github.com/user-attachments/assets/f76048d4-f741-41ec-9a56-f105da05756a)

1.  **Prometheus Operator**: A controller that manages Prometheus-related monitoring resources within Kubernetes. Users can define resources such as Prometheus and ServiceMonitor as Custom Resources (CRs), and the Operator automatically detects and updates configurations based on these definitions.
2.  **DaemonSet**: Responsible for automatically deploying exporters on each worker node. For Jetson Orin Nano nodes labeled with a specific tag such as `device=jetson`, a `nodeSelector` ensures that exporter Pods are scheduled accordingly. Even if a node restarts or a Pod is deleted, the DaemonSet ensures the exporter is automatically restored. Each worker node runs both the node-exporter, which collects general system metrics, and the jetson-exporter, which gathers Jetson-specific metrics such as GPU usage.
3.  **ServiceMonitor**: A Custom Resource managed by the Prometheus Operator, enabling Prometheus to automatically discover and scrape metrics from the exporters. Since each exporter runs in a Pod with a dynamically assigned IP, a corresponding Kubernetes Service is created to expose a stable access point via a fixed ClusterIP. The Service uses a label selector such as `app=jetson-exporter` to target the correct Pod. The Prometheus Operator continuously watches for ServiceMonitors, which in turn discover Services matching specific labels and configure Prometheus to scrape the `/metrics` endpoint of the associated exporter. The ServiceMonitor must include a label like `release=prometheus` to ensure it is recognized by the Operator.
4.  **Exporters**: Each Exporter exposes system metrics in a Prometheus-compatible format at the `/metrics` HTTP endpoint. Prometheus periodically scrapes these endpoints using a pull mechanism and stores all time-series data in its internal Time Series Database (TSDB).
5.  **Grafana**: Visualizes the data collected by Prometheus using PromQL queries. Users can monitor the real-time status of CPU, GPU, memory, and network usage for each node and the entire cluster, providing intuitive insights into resource utilization.

## ⚙️ Hardware & Environment

| Node Type | Device | OS |
| :--- | :--- | :--- |
| **Master Node** | Desktop | Ubuntu 22.04 |
| **Worker Node** | 2 × NVIDIA Jetson Orin Nano | JetPack 6.0 |
| **Worker Node** | 1 × Raspberry Pi 5 with Hailo | Raspberry Pi OS (64-bit) |

## 🛠️ Installation & Getting Started

Follow these steps to set up the Prometheus and Grafana dashboard:

-   **Step 1: Prometheus & Grafana**
    -   [📂 Prometheus & Grafana](https://github.com/jiiihwan/k8s-dashboard/tree/main/Prometheus%26Grafana)
-   **Step 2: Exporters** (GPU/NPU)
    -   [📂 Nvidia-exporter](https://github.com/jiiihwan/k8s-dashboard/tree/main/nvidia-exporter) (Desktop GPU)
    -   [🔗 Jetson_exporter](https://github.com/jiiihwan/jetson_exporter) (Orin Nano)
    -   [🔗 Hailo_exporter](https://github.com/jiiihwan/hailo_exporter) (RPi 5 NPU)

### 📚 References

Refer to the guides below if you need to set up Kubernetes or the default dashboard:

-   [📂 Kubernetes Setup Guide](https://github.com/jiiihwan/k8s-dashboard/tree/main/kubernetes)
-   [📂 K8s Dashboard Configuration Guide](https://github.com/jiiihwan/k8s-dashboard/tree/main/kubernetes/k8s_dashboard)

## 🔋 Dashboard Preview

![Grafana Dashboard](https://github.com/user-attachments/assets/f8c5a38a-8382-4edc-b511-a6b56bd2e01a)

-   **Top Row**: CPU, GPU, RAM, Network
-   **Left Column**: Cluster, Master Node, Worker Nodes
-   **Right Column**: Per-core CPU usage
-   **Chart**: Network usage ratio by node

**JSON Templates:**
-   [K8s Cluster Dashboard.json](https://github.com/jiiihwan/k8s-dashboard/blob/main/Prometheus%26Grafana/K8s%20Cluster%20Dashboard.json)
-   [K8s Cluster Dashboard with RPI.json](https://github.com/jiiihwan/k8s-dashboard/blob/main/Prometheus%26Grafana/K8s%20Cluster%20Dashboard%20with%20RPI.json)
