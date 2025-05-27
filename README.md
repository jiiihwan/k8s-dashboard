# k8s-dashboard
> ### k8s dashboard with real-time resource utilization
## Supported resoureces
  - CPU utilizaiton (%) 
    - with core utilizaion
  - GPU utilization (%)
  - RAM utilization (GB & %)
  - Network utilization (Bit/sec)
    - transmitted
    - received
   
## Environmental settings
- Master Node
  - Desktop (linux)
- Workder Node
  - 2 units of NVIDIA Jetson Orin Nano

---

# Installation
- There is specific guidelines in each folder

## k8s dashboard

  See [k8s_dashboard_installation.md](k8s_dashboard_installation.md) for details


## Prometheus-stack


  See [Prometheus-stack installation.md](installation.md) for details

## Exporters
- Used Jetson-exporter & Nvidia-exporter to export GPU usage

### Jetson-exporter
- Export GPU usage of jetson orin nano by using `jetson-stats`
- Optimized for automated installation and simplified management
  - It is inspired by how Node-exporter is installed via `prometheus-stack`

  See [jetson-exporter_installation.md](https://github.com/jiiihwan/k8s-dashboard/blob/main/exporter/jetson-exporter_installation.md) for details

### Nvidia-exporter
- export GPU usage of Nvidia GPU by using `nvidia-smi`

  See [[nvidia-exporter_installation.md](nvidia-exporter_installation.md)](https://github.com/jiiihwan/k8s-dashboard/blob/main/exporter/nvidia-exporter_installation.md) for details


---

## Grafana UI implementation
![스크린샷 2025-04-08 135758](https://github.com/user-attachments/assets/f8c5a38a-8382-4edc-b511-a6b56bd2e01a)

- In order from the top row : CPU, GPU, RAM, Network
- In order from the left column : Cluster, Master Node, Worker Node1, Worker Node2
- The rightmost one : Per-core CPU usage of orin nano
- A pie chart was placed in the cluster section of the network to show the usage ratio by node
