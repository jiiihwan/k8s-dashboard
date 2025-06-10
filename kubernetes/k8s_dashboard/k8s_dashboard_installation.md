# k8s dashboard installation
- k8s dashboard install guide using `helm`

---
### pod 보기
`kubectl get pods -A -o wide`

### 네임스페이스 지우기
`kubectl delete ns kubernetes-dashboard`

### 네임스페이스 안지워질때 써봐
`kubectl patch namespace kubernetes-dashboard -p '{"metadata":{"finalizers":[]}}' --type=merge`

`kubectl delete namespace kubernetes-dashboard --force --grace-period=0`

### Helm repo추가
`helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/`

### helm으로 대시보드 values.yaml 파일 다운받기
`helm show values kubernetes-dashboard/kubernetes-dashboard > values.yaml`

### values.yaml 수정하기

```yaml
nodeSelector:
  kubernetes.io/hostname: "<MASTER NODE NAME>" #마스터노드이름은 kubectl get nodes -o wide의 NAME영역
tolerations: #toleration은 이렇게 수정
- key: "node-role.kubernetes.io/control-plane"
  operator: "Exists"
  effect: "NoSchedule"
```


### 변경한 values 파일 이용해서 설치
`helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard -f values.yaml --namespace kubernetes-dashboard --create-namespace`

### helm으로 대시보드 7.5.0 버전 설치
`helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --version 7.5.0 --create-namespace --namespace kubernetes-dashboard`

### value+7.5.0 
`helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard -f values.yaml --version 7.5.0 --namespace kubernetes-dashboard --create-namespace`

### 대시보드 잘 깔렸는지 확인
`kubectl get svc -n kubernetes-dashboard`

### NodePort로 외부접속 설정
`kubectl get service kubernetes-dashboard-kong-proxy -n kubernetes-dashboard`

### helm 설치 기준 edit
`kubectl edit service kubernetes-dashboard-kong-proxy -n kubernetes-dashboard`

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
  type: NodePort #먼저수정
```

### 변경사항 확인
`kubectl get svc -n kubernetes-dashboard`

### 설치파일 생성
`vim dashboard-admin.yaml`

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

### 설치
`kubectl apply -f dashboard-admin.yaml`

### admin토큰값 생성
`kubectl -n kubernetes-dashboard create token admin-user`
`kubectl -n kubernetes-dashboard create token admin-user --duration 720h #토큰값 시간 길게`

<details>
  
<summary>대시보드 홈페이지 failed the initial dns/balancer resolve for 'kubernetes-dashboard-web' with: failed to receive reply from UDP server 10.96.0.10:53: timeout. 해결법 </summary>
`kubectl rollout restart deployment -n kube-system coredns`
</details>




# metrics-server installation
- namespace : 원래 kube-system에 설치되지만 kubernetes-dashboard namespace에 깔리도록 수정

## metrics-server의 yaml파일 작성
- 주요수정사항
  - 기본 설치 namespace가 kubernetes-dashboard
  - kube-system namespace의 metadata를 받아올수있게 함
  - 마스터노드에 깔리도록 NodeSelector 수정
- components.yaml 로 작성한다

[components.yaml](components.yaml) 참고

## 설치
```bash
kubectl apply -f components.yaml
kubectl get pods -A -o wide
kubectl get svc -n kubernetes-dashboard

#삭제하는법
kubectl delete -f components.yaml
```

