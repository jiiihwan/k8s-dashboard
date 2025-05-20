# Prometheus, Grafana  Helm이용 설치
- 다음 5개는 마스터노드에 설치한다
    - alertmanager
    - grafana
    - state metrics
    - prometheus operator
    - prometheus
---
### helm 설치
```
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

### Helm차트 repo 추가
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### monitoring 네임스페이스 생성
```
kubectl create namespace monitoring
```

### 노드 라벨링
- master node 에 key=master 라는 라벨링 추가
```
kubectl get nodes --show-labels
kubectl label nodes [node_name] key=master
```
### 프로메테우스 및 그라파나 설치 with NodeSelector
```
helm upgrade --install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring \
--set prometheus.prometheusSpec.nodeSelector.key=master \
--set prometheus.prometheusSpec.tolerations[0].key="node-role.kubernetes.io/control-plane" \
--set prometheus.prometheusSpec.tolerations[0].operator="Exists" \
--set prometheus.prometheusSpec.tolerations[0].effect="NoSchedule" \
--set alertmanager.alertmanagerSpec.nodeSelector.key=master \
--set alertmanager.alertmanagerSpec.tolerations[0].key="node-role.kubernetes.io/control-plane" \
--set alertmanager.alertmanagerSpec.tolerations[0].operator="Exists" \
--set alertmanager.alertmanagerSpec.tolerations[0].effect="NoSchedule" \
--set grafana.nodeSelector.key=master \
--set grafana.tolerations[0].key="node-role.kubernetes.io/control-plane" \
--set grafana.tolerations[0].operator="Exists" \
--set grafana.tolerations[0].effect="NoSchedule" \
--set prometheusOperator.nodeSelector.key=master \
--set prometheusOperator.tolerations[0].key="node-role.kubernetes.io/control-plane" \
--set prometheusOperator.tolerations[0].operator="Exists" \
--set prometheusOperator.tolerations[0].effect="NoSchedule" \
--set kube-state-metrics.nodeSelector.key=master \
--set kube-state-metrics.tolerations[0].key="node-role.kubernetes.io/control-plane" \
--set kube-state-metrics.tolerations[0].operator="Exists" \
--set kube-state-metrics.tolerations[0].effect="NoSchedule"
```
### 확인
`kubectl get svc -n monitoring`

### NodePort 수정
- 서비스 파일을 수정
- NodePort 이용하여 외부에서도 접속이 가능하게 한다
- Prometheus는 31001 포트, Grafana는 31002 포트를 사용했다

`kubectl edit svc prometheus-kube-prometheus-prometheus -n monitoring`
`kubectl edit svc prometheus-grafana -n monitoring`

```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: Service
metadata:
  annotations:
    meta.helm.sh/release-name: prometheus
    meta.helm.sh/release-namespace: monitoring
  creationTimestamp: "2025-03-23T11:20:46Z"
  labels:
    app: kube-prometheus-stack-prometheus
    app.kubernetes.io/instance: prometheus
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/part-of: kube-prometheus-stack
    app.kubernetes.io/version: 70.0.3
    chart: kube-prometheus-stack-70.0.3
    heritage: Helm
    release: prometheus
    self-monitor: "true"
  name: prometheus-kube-prometheus-prometheus
  namespace: monitoring
  resourceVersion: "3627847"
  uid: 18a37d54-e053-44e7-be03-1e7c86cfdb16
spec:
  clusterIP: 10.100.89.103
  clusterIPs:
  - 10.100.89.103
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: http-web
    nodePort: 31001 # 여기 추가
    port: 9090
    protocol: TCP
    targetPort: 9090
  - appProtocol: http
    name: reloader-web
    nodePort: 30319 
    port: 8080
    protocol: TCP
    targetPort: reloader-web
  selector:
    app.kubernetes.io/name: prometheus
    operator.prometheus.io/name: prometheus-kube-prometheus-prometheus
  sessionAffinity: None
  type: NodePort # 여기 이렇게 수정
status:
  loadBalancer: {}
```


### 리눅스에서 포트포워딩
```
sudo iptables -I INPUT -p tcp -s 0.0.0.0/0 -d 155.230.16.159 --dport 31001 -j ACCEPT
sudo iptables -I INPUT -p tcp -s 0.0.0.0/0 -d 155.230.16.159 --dport 31002 -j ACCEPT
```
### Helm values.yaml 수정
```
helm get values prometheus --namespace monitoring > values.yaml
helm upgrade prometheus prometheus-community/kube-prometheus-stack -f values.yaml --namespace monitoring
```
