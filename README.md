# 💻 k8s-dashboard
> ### k8s dashboard with real-time resource utilization

A custom dashboard implemented using Prometheus and Grafana to monitor real-time resource usage of nodes, including GPU utilization, which is not supported by the default Kubernetes dashboard.


### 📢 Supported resources
  1. **CPU** utilizaiton (%)
     - with core utilizaion
  3. **GPU** utilization (%)
  4. **RAM** utilization (GB & %)
  5. **Network** utilization (Bit/sec)
     - transmitted & received
   
### ⚙️ k8s environmental settings

| Node Type     | Device                          |
|---------------|----------------------------------|
| Master Node   | Desktop (Ubuntu 22.04)          |
| Worker Node   | 2 × NVIDIA Jetson Orin Nano     |



## 🧱 System Architecture
This system operates on a Kubernetes cluster, where Prometheus collects metrics from exporters installed on each worker node and visualizes them in real-time via Grafana. The system works as follows:

![스크린샷 2025-07-10 155012](https://github.com/user-attachments/assets/f76048d4-f741-41ec-9a56-f105da05756a)


1) Prometheus Operator is a controller that manages Prometheus-related monitoring resources within Kubernetes. Users can define resources such as Prometheus and ServiceMonitor as Custom Resources (CRs), and the Operator automatically detects and updates configurations based on these definitions.

2) DaemonSet is responsible for automatically deploying exporters on each worker node. For Jetson Orin Nano nodes labeled with a specific tag such as `device=jetson`, a `nodeSelector` ensures that exporter Pods are scheduled accordingly. Even if a node restarts or a Pod is deleted, the DaemonSet ensures the exporter is automatically restored. Each worker node runs both the node-exporter, which collects general system metrics, and the jetson-exporter, which gathers Jetson-specific metrics such as GPU usage.

3) A ServiceMonitor, a Custom Resource managed by the Prometheus Operator, enables Prometheus to automatically discover and scrape metrics from the exporters. Since each exporter runs in a Pod with a dynamically assigned IP, a corresponding Kubernetes Service is created to expose a stable access point via a fixed ClusterIP. The Service uses a label selector such as `app=jetson-exporter` to target the correct Pod. The Prometheus Operator continuously watches for ServiceMonitors, which in turn discover Services matching specific labels and configure Prometheus to scrape the `/metrics` endpoint of the associated exporter. The ServiceMonitor must include a label like `release=prometheus` to ensure it is recognized by the Operator.

4) Each Exporter exposes system metrics in a Prometheus-compatible format at the `/metrics` HTTP endpoint. Prometheus periodically scrapes these endpoints using a pull mechanism and stores all time-series data in its internal Time Series Database (TSDB).

5) Grafana visualizes the data collected by Prometheus using PromQL queries. Users can monitor the real-time status of CPU, GPU, memory, and network usage for each node and the entire cluster, providing intuitive insights into resource utilization.

## 🛠️ Installation

There is specific guidelines in each folder

### 1. kubernetes

See [kubernetes_README.md](https://github.com/jiiihwan/k8s-dashboard/tree/main/kubernetes) for details

### 2. k8s dashboard

See [k8s_dashboard_README.md](https://github.com/jiiihwan/k8s-dashboard/tree/main/kubernetes/k8s_dashboard) for details

### 3. Prometheus & Grafana

See [prometheus_stack_README.md](https://github.com/jiiihwan/k8s-dashboard/blob/main/Prometheus&Grafana/prometheus_stack_installation.md) for details

### 4. Exporters

Used Jetson-exporter & Nvidia-exporter to export GPU usage

#### 4.1 Nvidia-exporter
- export GPU usage of Nvidia GPU by using `nvidia-smi`

See [nvidia-exporter_README.md](https://github.com/jiiihwan/k8s-dashboard/blob/main/exporter/nvidia-exporter_installation.md) for details

#### 4.2 Jetson_exporter
- Export GPU usage of jetson orin nano by using `jtop`
- Optimized for automated installation and simplified management
  - It is inspired by how Node-exporter is installed via `prometheus-stack`

See https://github.com/jiiihwan/jetson_exporter for details

#### 4.3 Hailo_exporter
- Export NPU usage of Hailo on Raspbarry pi 5 by analyzing protobuf binary file.
- Also optimized for automated installation and simplified management]

See https://github.com/jiiihwan/hailo_exporter for details

---

## 🔋 Grafana UI implementation
![스크린샷 2025-04-08 135758](https://github.com/user-attachments/assets/f8c5a38a-8382-4edc-b511-a6b56bd2e01a)

- In order from the top row : CPU, GPU, RAM, Network
- In order from the left column : Cluster, Master Node, Worker Node1, Worker Node2
- The rightmost one : Per-core CPU usage of orin nano
- A pie chart was placed in the cluster section of the network to show the usage ratio by node
