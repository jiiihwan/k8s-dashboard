# 🔧 트러블슈팅: Node Exporter 수집 문제

[**English**](problem_solving.en.md) | [**한국어**](problem_solving.md)

## 🔴 문제 상황 (Issue)

**증상**:
- 마스터 노드의 Node Exporter는 정상적으로 동작하지만, **Jetson Orin Nano (워커 노드)** 의 Node Exporter 메트릭이 Prometheus에서 수집되지 않음.
- 로그 분석 결과, CPU Frequency 수집 중 오류가 발생하거나 파일 크기 관련 경고가 지속됨.
- daemonset의 yaml에서 cpufreq수집을 비활성화 했음에도 해결되지 않음.
-  node-exporter 컨피그맵yml수정, node-exporter 재설치 등의 방법에도 해결되지 않음.

**원인**:
- Jetson Orin Nano의 `thermal_zone` 컬렉터와 관련된 호환성 문제로 추정됨. (관련 이슈: [GitHub Issue #3071](https://github.com/prometheus/node_exporter/issues/3071))

---

## 🟢 해결 방법 (Solution)

Node Exporter의 실행 인자에 `--no-collector.thermal_zone` 옵션을 추가하여 해당 컬렉터를 비활성화합니다.

### DaemonSet 수정

1. Node Exporter의 DaemonSet 수정 모드로 진입합니다.
    ```bash
    kubectl edit daemonset prometheus-prometheus-node-exporter -n monitoring
    ```

2. `containers` 섹션의 `args`에 옵션을 추가합니다.

    ```yaml
    spec:
      containers:
      - name: node-exporter
        args:
        - --path.procfs=/host/proc
        - --path.sysfs=/host/sys
        - --path.rootfs=/host/root
        - --no-collector.thermal_zone  # <-- 이 줄을 추가하세요
    ```

3. 저장하고 닫으면 DaemonSet이 파드를 재생성하며 정상적으로 수집이 시작됩니다.
