# 프로메테우스-node exporter 수집안되는문제
- 워커노드는 orin nano. 이때 마스터노드는 되는데 워커노드의 node exporter가 프로메테우스에서 수집안되는 문제가 있었다
- 로그 분석했을때 cpu freqency 를 수집할때 파일이 너무 크다는 로그가 계속해서 떠서 데몬셋 yml설정에서 `cpufreq`수집을 비활성화함
- 그런데도 안됐다. 그래서 대략 3일정도를 찾아본결과….
    - 프로메테우스 연동을 위한 node-exporter 컨피그맵yml수정
    - node-exporter 재설치 등등 뻘짓을 많이했다
- 혹시나 node-exporter의 문제가 아니라 오린나노의 특성문제가 아닐까 해서 찾아봤는데 결국 [Github](https://github.com/prometheus/node_exporter/issues/3071)에서 같은 증상을 보이는 사람을 찾음
    - 어떻게 했는지는 몰라도 `thermal zone` 을 원인으로 좁혔다는데 아무튼 이걸 수집 안하도록 데몬셋yml설정했더니 이제 정상적으로 수집된다…..!
 
## 해결방법
- node-exporter 의 daemon set 수정

`kubectl edit daemonset prometheus-prometheus-node-exporter -n monitoring`

```yaml
~
containers:
        - args:
            - --path.procfs=/host/proc
            - --path.sysfs=/host/sys
            - --path.rootfs=/host/root
            - --no-collector.thermal_zone #추가
~
```

