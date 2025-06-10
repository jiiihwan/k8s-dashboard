# kubernetes 설치
먼저 쿠버네티스 설치를 한다.

### Swap 비활성화
```bash
sudo swapoff -a
```

### 네트워크 모듈 로드
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```
### sysctl 파라미터 설정
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

### sysctl 파라미터 적용
```bash
sudo sysctl --system
```

### containerd 설치
```bash
sudo apt install -y containerd
```


### containerd 설정 파일 생성 및 수정
```bash
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo vim /etc/containerd/config.toml
```
`SystemdCgroup = true` 로 수정 후 저장

### containerd 서비스 재시작
```bash
sudo systemctl restart containerd.service
```

### 서비스 상태 확인
```bash
sudo systemctl status containerd.service
```
### K8s 설치를 위한 패키지 설정

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
```

```bash
sudo mkdir -p /etc/apt/keyrings
```

```bash
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

```bash
sudo apt-get update
```

### 쿠버네티스 설치
```bash
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
이후에 워커노드를 join시키기 위해 필요하니 출력된 join명령어를 어딘가에 메모해두기. 

### 버전확인
```bash
kubelet --version
kubeadm version
kubectl version --output=yaml
```

### (마스터노드만) CNI - Flannel 설치 
```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

### (워커노드) join
메모해놓은 join명령어를 그대로 워커노드에서 복붙한다
```bash
kubeadm join <IP:PORT> --token <token> --discovery-token-ca-cert-hash sha256:<hash_number>
```

### 마스터노드에서 노드 확인
```bash
sudo kubectl get node
```

### tip) join키 다시 발급받기
만약 join키를 잃어버려다면 마스터노드에서 다시 발급받을 수 있다.
```bash
kubeadm token create --print-join-command
```
