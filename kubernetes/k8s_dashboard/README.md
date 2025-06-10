# 🛠 k8s dashboard installation
k8s dashboard install guide using `helm`

## 📎Helm?
> 쿠버네티스(Kubernetes) 환경에서 애플리케이션과 관련 리소스들을 쉽고 일관되게 배포, 관리, 업그레이드, 롤백할 수 있도록 도와주는 오픈소스 패키지 매니저

리눅스에 apt, python에 pip가 있다면, 쿠버네티스에는 helm이 있다고 생각하면 편하다.

### Helm Chart
> 쿠버네티스 애플리케이션을 배포하는 데 필요한 리소스(Deployment, Service, ConfigMap 등)를 정의한 파일들의 집합
- 차트에는 애플리케이션의 메타데이터(Chart.yaml), 기본 설정값(values.yaml), 쿠버네티스 리소스 템플릿(templates 디렉토리) 등이 포함되어 있다.
- 이 Helm Chart를 통해 복잡한 애플리케이션도 손쉽게 설치하고 관리할 수 있다.

### Helm Repository
> 여러 개의 차트(Chart)를 저장하고 공유하는 공간
- 차트들을 모아놓은 저장소로, apt의 패키지 저장소나 Docker Hub와 비슷한 역할을 한다.
  
---

## k8s dashboard 설치

### 📦 1. Helm repository 추가

```bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
```

### 🫵 2. nodeSelector로 node 선택
기본적으로 아무것도 설정하지 않는다면 마스터노드에는 pod가 올라가지 않고 무작위의 워커노드에 pod가 올라가게 된다.
마스터노드에 k8s dashboard를 설치할 것이므로 nodeSelector를 통해 node를 선택해준다.

#### 대시보드 values.yaml 파일 다운로드

```bash
helm show values kubernetes-dashboard/kubernetes-dashboard > values.yaml
```

#### pod 보기
```bash
kubectl get pods -A -o wide
```

#### values.yaml 수정
```bash
vim values.yaml
```

```yaml
nodeSelector:
  kubernetes.io/hostname: "<MASTER NODE NAME>" #마스터노드이름은 kubectl get nodes -o wide의 NAME영역
tolerations: #toleration은 이렇게 수정
- key: "node-role.kubernetes.io/control-plane"
  operator: "Exists"
  effect: "NoSchedule"
```

### 🔨 3. 변경한 values 파일 이용해서 설치
kubernetes-dashboard 라는 네임스페이스를 생성해서 그곳에 설치한다.

```bash
helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard -f values.yaml --version 7.5.0 --namespace kubernetes-dashboard --create-namespace
```

#### dashbord가 잘 설치되었는지 확인
```bash
kubectl get svc -n kubernetes-dashboard
```

### 🔗 4. NodePort로 외부접속 설정
NodePort는 쿠버네티스(Kubernetes)에서 서비스(Service)를 외부에 노출하는 방법 중 하나이다.

k8s dashboard 페이지를 마스터노드 뿐만 아니라 외부에서도 볼 수 있게 하기 위해서 외부접속 설정을 한다.
open하는 포트는 여기서는 31000포트를 사용했다.

#### 설치된 서비스 확인
```bash
kubectl get service kubernetes-dashboard-kong-proxy -n kubernetes-dashboard
```

#### 서비스 edit
```
kubectl edit service kubernetes-dashboard-kong-proxy -n kubernetes-dashboard
```

`type: NodePort` 를 입력하고 `:wq` 입력 후에 다시 명령어를 입력해서 `nodePort: <port>` 가 생기면 포트를 바꿔주도록 하자

한번에 바꾸면 에러가 나는 경우가 잦다.

```yaml
ports:
  - name: kong-proxy-tls
    nodePort: 31000 #나중에 수정
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    app.kubernetes.io/component: app
    app.kubernetes.io/instance: kubernetes-dashboard
    app.kubernetes.io/name: kong
  sessionAffinity: None
  type: NodePort #먼저 수정
```

#### 변경사항 확인
```
kubectl get svc -n kubernetes-dashboard
```

### 🗝️ 5. admin권한을 위한 파일 생성
admin-user 서비스 어카운트에 관리자권한을 부여한다.

이 서비스 어카운트로 토큰 인증 시 모든 네임스페이스와 리소스에 접근할 수 있게 된다.

```
vim dashboard-admin.yaml
```

### dashboard-admin.yaml 입력

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

#### 설치
```
kubectl apply -f dashboard-admin.yaml
```

#### admin토큰값 생성

```
kubectl -n kubernetes-dashboard create token admin-user
```
#### tip) 토큰시간 길게 설정하기
기본 토큰 지속시간이 짧아서 다시 발급받아야 하는 경우가 많다. 이 때 `--duration` 옵션으로 시간을 길게 지정할 수 있다
```
kubectl -n kubernetes-dashboard create token admin-user --duration 720h #토큰값 시간 길게
```

### 🗑️tip) 네임스페이스 지우기
잘못 설치했다면 네임스페이스 지우면 관련 설정이 모두 지워진다
```bash
kubectl delete ns kubernetes-dashboard
```
#### tip) 네임스페이스 안지워질때 
```bash
kubectl patch namespace kubernetes-dashboard -p '{"metadata":{"finalizers":[]}}' --type=merge
```
```bash
kubectl delete namespace kubernetes-dashboard --force --grace-period=0
```

# 🛠️ metrics-server installation
## metrics-server?
> Kubernetes 클러스터 내에서 각 노드와 파드(Pod)의 리소스 사용량(CPU, 메모리 등) 데이터를 수집·집계하여, 이를 Kubernetes API 서버에 제공하는 경량화된 서비스

쿠버네티스 대쉬보드에서 cpu와 메모리사용량을 보려면 metrics-server를 설치해야한다

이때 namespace는 원래 `kube-system`에 설치되지만 이전에 dashboard를 설치한 곳에 통합적으로 설치하기 위해 `kubernetes-dashboard` namespace에 깔리도록 수정한다.

### 📄 metrics-server의 yaml파일 작성
주요수정사항
- 기본 설치 namespace가 `kubernetes-dashboard`
- `kube-system` namespace의 metadata를 받아올수있게 함
- 마스터노드에 깔리도록 `nodeSelector` 수정
- components.yaml 로 작성한다

[components.yaml](components.yaml) 참고

### 🔨 설치 및 확인
```bash
kubectl apply -f components.yaml
kubectl get pods -A -o wide
kubectl get svc -n kubernetes-dashboard
```

