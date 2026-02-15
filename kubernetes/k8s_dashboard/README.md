# 🛠️ Kubernetes Dashboard 설치 가이드

[**English**](README.en.md) | [**한국어**](README.md)

이 문서는 Helm을 사용하여 Kubernetes Dashboard를 설치하고 구성하는 방법을 안내합니다.

## 📋 개요 (Overview)

Kubernetes Dashboard는 컨테이너화된 애플리케이션을 배포하고 문제를 해결하며, 클러스터 리소스를 관리할 수 있는 웹 기반 UI입니다.

> **Helm**이란?
> Kubernetes 애플리케이션의 배포 및 관리를 돕는 패키지 매니저입니다. (Linux의 `apt`, Python의 `pip`와 유사)

---

## 🚀 설치 단계 (Installation Steps)

### 1. Helm Repository 추가

```bash
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
```

### 2. 설정 파일 (values.yaml) 준비

마스터 노드에만 Dashboard가 배포되도록 설정하기 위해 `values.yaml`을 수정합니다.

#### 2.1 기본값 다운로드
```bash
helm show values kubernetes-dashboard/kubernetes-dashboard > values.yaml
```

#### 2.2 values.yaml 수정
`nodeSelector`와 `tolerations`를 추가하여 마스터 노드(Control Plane)에 파드가 생성되도록 합니다.

```yaml
nodeSelector:
  kubernetes.io/hostname: "<MASTER_NODE_NAME>" # 마스터 노드 이름 확인: kubectl get nodes
tolerations:
- key: "node-role.kubernetes.io/control-plane"
  operator: "Exists"
  effect: "NoSchedule"
```

### 3. Helm을 이용한 설치

`kubernetes-dashboard` 네임스페이스를 생성하고 설치를 진행합니다.

```bash
helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard \
  -f values.yaml \
  --version 7.5.0 \
  --namespace kubernetes-dashboard \
  --create-namespace
```

설치 확인:
```bash
kubectl get svc -n kubernetes-dashboard
```

---

## 🌐 외부 접속 설정 (NodePort)

외부 브라우저에서 대시보드에 접근할 수 있도록 `NodePort` 서비스를 설정합니다.

1. 서비스 수정 모드 진입:
    ```bash
    kubectl edit service kubernetes-dashboard-kong-proxy -n kubernetes-dashboard
    ```
2. `type: NodePort` 및 `nodePort: 31000` 설정:

    ```yaml
    spec:
      ports:
      - name: kong-proxy-tls
        nodePort: 31000  # 접속 포트 지정
        port: 443
        protocol: TCP
        targetPort: 8443
      type: NodePort     # ClusterIP -> NodePort 변경
    ```

이제 `https://<Master_IP>:31000`으로 접속할 수 있습니다.

---

## 🔑 관리자 권한 설정 (Admin User)

모든 네임스페이스에 접근할 수 있는 관리자 계정을 생성합니다.

### 1. ServiceAccount 및 RoleBinding 생성

`dashboard-admin.yaml` 파일을 작성합니다.

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

적용:
```bash
kubectl apply -f dashboard-admin.yaml
```

### 2. 로그인 토큰 생성
대시보드 로그인에 사용할 토큰을 생성합니다 (`--duration` 옵션으로 유효기간 연장 가능).

```bash
kubectl -n kubernetes-dashboard create token admin-user --duration 720h
```

---

## 📊 Metrics Server 설치 (리소스 모니터링)

대시보드에서 CPU 및 메모리 사용량을 확인하려면 **Metrics Server**가 필요합니다.

### 1. 구성 요소 파일 (components.yaml)
제공된 [components.yaml](components.yaml) 파일은 다음 사항이 수정되어 있습니다:
- **Namespace**: `kubernetes-dashboard`에 통합 설치
- **NodeSelector**: 마스터 노드에 배포
- **Kubelet 통신**: `kube-system` 메타데이터 접근 허용

### 2. 설치 및 확인

```bash
kubectl apply -f components.yaml
```

설치 후 `kubectl top nodes` 명령어로 메트릭 수집을 확인할 수 있습니다.

---

### 💡 유용한 팁 (Tips)

- **네임스페이스 삭제**: 설치를 완전히 초기화하려면 네임스페이스를 삭제하세요.
  ```bash
  kubectl delete ns kubernetes-dashboard
  ```
- **강제 삭제**: 삭제가 멈출 경우 Finalizer를 제거합니다.
  ```bash
  kubectl patch namespace kubernetes-dashboard -p '{"metadata":{"finalizers":[]}}' --type=merge
  ```
