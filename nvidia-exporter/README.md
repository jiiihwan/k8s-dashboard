# 🛠️ Nvidia Exporter 설치 가이드

[**English**](README.en.md) | [**한국어**](README.md)

이 문서는 **NVIDIA GPU**를 사용하는 노드의 GPU 사용량을 모니터링하기 위해 `nvidia_gpu_exporter`를 설치하고 시스템 서비스로 등록하는 방법을 안내합니다.

> **참고**: [nvidia_gpu_exporter](https://github.com/utkuozdemir/nvidia_gpu_exporter)는 `nvidia-smi` 바이너리를 이용하여 GPU 메트릭을 수집하는 Go 언어 기반의 Exporter입니다.

## 🚀 설치 단계 (Installation Steps)

### 1. 버전 설정 및 다운로드

[릴리즈 페이지](https://github.com/utkuozdemir/nvidia_gpu_exporter/releases)에서 최신 버전을 확인하고 환경변수를 설정합니다.

```bash
VERSION=1.3.0
wget https://github.com/utkuozdemir/nvidia_gpu_exporter/releases/download/v${VERSION}/nvidia_gpu_exporter_${VERSION}_linux_x86_64.tar.gz
```

### 2. 압축 해제 및 설치

```bash
tar -xvzf nvidia_gpu_exporter_${VERSION}_linux_x86_64.tar.gz
sudo mv nvidia_gpu_exporter /usr/bin/
```

### 3. 시스템 사용자 생성

보안을 위해 전용 시스템 사용자를 생성합니다.

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin nvidia_gpu_exporter
```

---

## ⚙️ 서비스 등록 (Systemd Service)

Nvidia Exporter를 백그라운드 서비스로 실행하기 위해 systemd 유닛 파일을 생성합니다.

### 1. 서비스 파일 생성

```bash
sudo vim /etc/systemd/system/nvidia_gpu_exporter.service
```

다음 내용을 붙여넣으세요:

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

### 2. 서비스 활성화 및 시작

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now nvidia_gpu_exporter
```

### 3. 상태 확인

```bash
sudo systemctl status nvidia_gpu_exporter
```

정상적으로 실행 중이라면 `Active: active (running)` 상태가 표시됩니다.

---

## 🐳 Docker/Pod로 배포 (Alternative)

Systemd 서비스 대신, Docker 이미지를 생성하여 Kubernetes **Pod** 또는 **DaemonSet**으로 배포할 수도 있습니다.

1.  **Dockerfile 작성**:
    ```dockerfile
    FROM nvidia/cuda:12.0.0-base-ubuntu22.04
    COPY nvidia_gpu_exporter /usr/bin/nvidia_gpu_exporter
    ENTRYPOINT ["/usr/bin/nvidia_gpu_exporter"]
    ```
2.  **이미지 빌드 및 푸시**: 이미지를 빌드하여 레지스트리에 푸시합니다.
3.  **Kubernetes 배포**: 해당 이미지를 사용하는 Pod 또는 DaemonSet yaml 파일을 작성하여 배포합니다.

