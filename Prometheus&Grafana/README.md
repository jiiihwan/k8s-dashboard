# ⚙️ Prometheus & Grafana 설치 가이드

[**English**](README.en.md) | [**한국어**](README.md)

이 문서는 Helm을 사용하여 Prometheus와 Grafana를 설치하고 설정하는 방법을 설명합니다.
모니터링 스택의 주요 구성 요소(Alertmanager, Grafana, Prometheus 등)는 **마스터 노드**에 설치됩니다.

## 🚀 설치 단계 (Installation Steps)

### 1. Helm 설치 및 Repo 추가

Helm이 설치되어 있지 않다면 먼저 설치합니다.
```bash
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
```

Helm 저장소를 추가하고 업데이트합니다.
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### 2. 네임스페이스 생성 및 노드 라벨링

모니터링 전용 네임스페이스를 생성합니다.
```bash
kubectl create namespace monitoring
```

마스터 노드에 `key=master` 라벨을 추가하여, 모니터링 파드들이 마스터 노드에 배치되도록 준비합니다.
```bash
kubectl label nodes <MASTER_NODE_NAME> key=master
# 확인: kubectl get nodes --show-labels
```

### 3. Prometheus Stack 설치

`kube-prometheus-stack`을 설치합니다. 아래 명령어는 `nodeSelector`를 사용하여 주요 컴포넌트를 마스터 노드에 배치하는 설정입니다.

> **참고**: `values.yaml` 파일을 사용하여 더 상세한 설정을 적용하는 것이 권장됩니다. 상세 내용은 [Helm 설정 가이드](helm_setting.md)를 참고하세요.

```bash
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

설치 확인:
```bash
kubectl get svc -n monitoring
```

---

## 🌐 외부 접속 설정 (NodePort)

외부에서 Prometheus와 Grafana에 접속할 수 있도록 Service를 NodePort 타입으로 변경합니다.
(기본 포트: Prometheus `31001`, Grafana `31002` 사용 예시)

### 1. Prometheus 서비스 수정
```bash
kubectl edit svc prometheus-kube-prometheus-prometheus -n monitoring
```
```yaml
spec:
  type: NodePort
  ports:
    - name: http-web
      port: 9090
      nodePort: 31001  # 포트 지정
```

### 2. Grafana 서비스 수정
```bash
kubectl edit svc prometheus-grafana -n monitoring
```
```yaml
spec:
  type: NodePort
  ports:
    - name: http-web
      port: 80
      nodePort: 31002  # 포트 지정
```

### 3. 포트 포워딩 (Linux/IPTables 사용 시)
필요한 경우 iptables를 사용하여 포트를 개방합니다.
```bash
sudo iptables -I INPUT -p tcp --dport 31001 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 31002 -j ACCEPT
```

---

## 📚 추가 가이드

- **[Helm 설정 상세 가이드](helm_setting.ko.md)**: `values.yaml`을 이용한 고급 설정 방법
- **[트러블슈팅 가이드](problem_solving.md)**: Node Exporter 수집 문제 해결 (Jetson Orin Nano)
