# helm values.yaml
- helm으로 설치할 때 설정값들을 적용하는 파일
- prometheus-stack의 원본 values.yaml은 2000줄이 넘어 수정하기 힘들다
- 그래서 USER-SUPPLIED VALUES: 옵션으로 필요한 부분만 적고 `helm upgrade`로 적용시켜준다

## 주요 수정점
1. nodeSelector로 워커노드말고 마스터 노드에 prometheus-stack의 요소들이 설치되도록 함
2. grafana의 자동 새로고침 주기는 원래 최소가 5초 이지만 1초로도 설정 가능하게 함
3. prometheus가 master node의 nvidia-exporter를 감지할 수 있게 `additionalScrapeConfigs` 설정
   - 추가로 수집주기는 1초로 설정
4. node-exporter의 수집주기를 1초로 설정

### values.yaml

```yaml
USER-SUPPLIED VALUES:
alertmanager:
  alertmanagerSpec:
    nodeSelector:
      key: master
    tolerations:
    - effect: NoSchedule
      key: node-role.kubernetes.io/control-plane
      operator: Exists
grafana:
  grafana.ini:
    dashboards:
      min_refresh_interval: 1s
  nodeSelector:
    key: master
  tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/control-plane
    operator: Exists
kube-state-metrics:
  nodeSelector:
    key: master
  tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/control-plane
    operator: Exists
prometheus:
  prometheusSpec:
    nodeSelector:
      key: master
    tolerations:
    - effect: NoSchedule
      key: node-role.kubernetes.io/control-plane
      operator: Exists
    additionalScrapeConfigs:
    - job_name: "nvidia_gpu_exporter"
      scrape_interval: 1s
      static_configs:
      - targets:
        - "<IP>:<PORT>"
prometheus-node-exporter:
  enabled: true
  serviceMonitor:
    interval: 1s
    scrapeTimeout: 1s
prometheusOperator:
  nodeSelector:
    key: master
  tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/control-plane
    operator: Exists
```

### 작성 후 적용
`helm upgrade prometheus prometheus-community/kube-prometheus-stack -n monitoring -f values.yaml`
