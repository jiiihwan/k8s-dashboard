# 🛠️ Nvidia-exporter installation
- https://github.com/utkuozdemir/nvidia_gpu_exporter 참고
  - nvidia-smi의 바이너리를 이용해서 메트릭을 수집하는 go lang으로 작성된 exporter 
- jetson-exporter와 다르게 master node의 일반 Nvidia GPU의 사용량 관찰용이기 때문에 리눅스 백그라운드 서비스로 동작하게 했다.

## 설치
아래 내용은 위 참고 링크의 [리눅스 서비스 설치방법](https://github.com/utkuozdemir/nvidia_gpu_exporter/blob/master/INSTALL.md)을 그대로 설명한 것이다.

### 버전 변수 설정
```bash
VERSION=1.3.0
```

### Linux binary 파일 다운로드
```
wget https://github.com/utkuozdemir/nvidia_gpu_exporter/releases/download/v${VERSION}/nvidia_gpu_exporter_${VERSION}_linux_x86_64.tar.gz
```

### 압축해제
```bash
tar -xvzf nvidia_gpu_exporter_${VERSION}_linux_x86_64.tar.gz
```

### /usr/bin 디렉토리로 이동
```bash
mv nvidia_gpu_exporter /usr/bin
```

### nvidia_gpu_exporter 라는 이름의 system user and group 만들기 
```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin nvidia_gpu_exporter
cd /etc/systemd/system
```

### 서비스 파일 생성
```bash
vim nvidia_gpu_exporter.service
```

아래 내용을 그대로 붙혀넣기
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

### 데몬 리로드
```bash
sudo systemctl daemon-reload
```

### 활성화 및 설치 확인

```
sudo systemctl enable --now nvidia_gpu_exporter
sudo systemctl status nvidia_gpu_exporter
```
