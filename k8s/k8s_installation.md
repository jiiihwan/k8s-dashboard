```# Swap 비활성화
sudo swapoff -a

# 네트워크 모듈 로드
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl 파라미터 설정
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF


# sysctl 파라미터 적용
sudo sysctl --system

# containerd 설치
sudo apt install -y containerd

# containerd 설정 파일 생성 및 수정
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo vim /etc/containerd/config.toml
# SystemdCgroup = true 로 수정 후 저장

# containerd 서비스 재시작
sudo systemctl restart containerd.service

# 서비스 상태 확인
sudo systemctl status containerd.service

# K8s 설치를 위한 패키지 설정
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

sudo mkdir -p /etc/apt/keyrings

sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

#쿠버네티스 설치
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

#버전확인
kubelet --version
kubeadm version
kubectl version --output=yaml

#(마스터노드만) CNI - Flannel 설치 
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

#(워커노드) join
kubeadm join 155.230.16.159:6443 --token w6lwjt.c619ffa27jpaapi7 --discovery-token-ca-cert-hash sha256:711154c4395ebba31ecbd2f0816609f066ab3278a59b7b8185991a1b84b27753 

#마스터노드에서 노드 확인
sudo kubectl get node

#join키 다시 발급받기
kubeadm token create --print-join-command

```
