# 🛠️ Nvidia Exporter Installation Guide

[**English**](README.md) | [**한국어**](README.ko.md)

This guide explains how to install `nvidia_gpu_exporter` and register it as a system service to monitor Nvidia GPU usage on **nodes using NVIDIA GPUs**.

> **Reference**: [nvidia_gpu_exporter](https://github.com/utkuozdemir/nvidia_gpu_exporter) is a Go-based exporter that collects GPU metrics using the `nvidia-smi` binary.

## 🚀 Installation Steps

### 1. Set Version & Download

Check the latest version on the [Release Page](https://github.com/utkuozdemir/nvidia_gpu_exporter/releases) and set the variable.

```bash
VERSION=1.3.0
wget https://github.com/utkuozdemir/nvidia_gpu_exporter/releases/download/v${VERSION}/nvidia_gpu_exporter_${VERSION}_linux_x86_64.tar.gz
```

### 2. Extract & Install

```bash
tar -xvzf nvidia_gpu_exporter_${VERSION}_linux_x86_64.tar.gz
sudo mv nvidia_gpu_exporter /usr/bin/
```

### 3. Create System User

Create a dedicated system user for security.

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin nvidia_gpu_exporter
```

---

## ⚙️ Register Systemd Service

Create a systemd unit file to run Nvidia Exporter as a background service.

### 1. Create Service File

```bash
sudo vim /etc/systemd/system/nvidia_gpu_exporter.service
```

Paste the following content:

```ini
[Unit]
Description=Nvidia GPU Exporter
After=network-online.target

[Service]
Type=simple

User=nvidia_gpu_exporter
Group=nvidia_gpu_exporter

ExecStart=/usr/bin/nvidia_gpu_exporter

SyslogIdentifier=nvidia_gpu_exporter

Restart=always
RestartSec=1

[Install]
WantedBy=multi-user.target
```

### 2. Enable & Start Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now nvidia_gpu_exporter
```

### 3. Verify Status

```bash
sudo systemctl status nvidia_gpu_exporter
```

You should see `Active: active (running)`.

---

## 🐳 Deploy as Docker/Pod (Alternative)

Instead of using a Systemd service, you can build a Docker image and deploy it as a Kubernetes **Pod** or **DaemonSet**.

1.  **Create Dockerfile**:
    ```dockerfile
    FROM nvidia/cuda:12.0.0-base-ubuntu22.04
    COPY nvidia_gpu_exporter /usr/bin/nvidia_gpu_exporter
    ENTRYPOINT ["/usr/bin/nvidia_gpu_exporter"]
    ```
2.  **Build & Push**: Build the image and push it to your registry.
3.  **Deploy**: Create a YAML file for a Pod or DaemonSet using this image and deploy it to your cluster.

