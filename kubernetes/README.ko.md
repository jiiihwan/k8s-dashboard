# 🛠️ Kubernetes 설치 가이드

[**English**](README.md) | [**한국어**](README.ko.md)

이 문서는 Kubernetes 클러스터 설치 및 초기 설정 방법을 안내합니다.

## 🔧 환경 구성 (Environment Setup)

| 노드 유형 | 장비 | OS |
| :--- | :--- | :--- |
| **Master Node** | Desktop | Ubuntu 22.04 |
| **Worker Node** | 2 × NVIDIA Jetson Orin Nano | JetPack 6.0 |

> **Note**: Container Runtime은 `containerd`를 사용합니다.

### 1. 사전 요구 사항 (Prerequisites)

모든 노드(마스터 및 워커)에서 다음 명령어를 실행하여 환경을 설정합니다.

#### 1.1 Swap 비활성화
안정적인 Kubernetes 동작을 위해 스왑 메모리를 비활성화합니다.
```bash
sudo swapoff -a
```

#### 1.2 네트워크 모듈 로드
필요한 커널 모듈을 로드합니다.
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

#### 1.3 Sysctl 파라미터 설정
네트워크 브리지 설정을 적용합니다.
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### 2. Container Runtime 설치 (containerd)

```bash
sudo apt install -y containerd
```

#### 설정 파일 생성 및 수정
`SystemdCgroup`을 활성화하기 위해 설정을 수정합니다.
```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
```

`/etc/containerd/config.toml` 파일을 열어 `SystemdCgroup` 값을 `true`로 변경합니다.
```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```

서비스를 재시작합니다.
```bash
sudo systemctl restart containerd.service
sudo systemctl status containerd.service
```

---

## 📦 Kubernetes 설치

### 1. 패키지 저장소 설정
Kubernetes 패키지 저장소를 추가합니다.

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo mkdir -p /etc/apt/keyrings

sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
```

### 2. 도구 설치 (kubelet, kubeadm, kubectl)

```bash
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 3. 클러스터 초기화 (Master Node Only)

마스터 노드에서 다음 명령어로 클러스터를 초기화합니다.
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
> **중요**: 초기화 후 출력되는 `kubeadm join` 명령어를 복사해 두세요. 워커 노드를 연결할 때 필요합니다.

#### 사용자 접근 권한 설정
`root`가 아닌 일반 사용자도 `kubectl`을 사용할 수 있도록 설정합니다.
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### CNI (Container Network Interface) 설치
네트워크 플러그인으로 Flannel을 설치합니다.
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### 4. 워커 노드 연결 (Worker Node)

마스터 노드 초기화 시 저장해둔 `kubeadm join` 명령어를 워커 노드에서 실행합니다.

```bash
sudo kubeadm join <Master_IP>:<Port> --token <Token> --discovery-token-ca-cert-hash sha256:<Hash>
```

> **Tip**: 토큰을 분실한 경우 마스터 노드에서 다음 명령어로 재발급 받을 수 있습니다.
> ```bash
> kubeadm token create --print-join-command
> ```


