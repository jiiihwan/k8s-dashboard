# Nvidia-exporet installation
- https://github.com/utkuozdemir/nvidia_gpu_exporter 참고
- jetson-exporter와 다르게 desktop master node용이기 때문에 리눅스 백그라운드 서비스로 동작한다

```bash
VERSION=1.3.0
wget https://github.com/utkuozdemir/nvidia_gpu_exporter/releases/download/v${VERSION}/nvidia_gpu_exporter_${VERSION}_linux_x86_64.tar.gz
tar -xvzf nvidia_gpu_exporter_${VERSION}_linux_x86_64.tar.gz
mv nvidia_gpu_exporter /usr/bin
nvidia_gpu_exporter --help
```
```
sudo useradd --system --no-create-home --shell /usr/sbin/nologin nvidia_gpu_exporter
cd /etc/systemd/system
```

### 서비스 파일 수정
`vim nvidia_gpu_exporter.service`

```
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

`sudo systemctl daemon-reload`


### 활성화 및 설치 확인

```
sudo systemctl enable --now nvidia_gpu_exporter
sudo systemctl status nvidia_gpu_exporter
```
