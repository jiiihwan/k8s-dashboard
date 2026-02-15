# 🛠️ Helm 설정 가이드 (values.yaml)

[**English**](helm_setting.md) | [**한국어**](helm_setting.ko.md)

Helm을 사용하여 `kube-prometheus-stack`을 설치할 때, `values.yaml` 파일을 수정하여 사용자 정의 설정을 적용할 수 있습니다.
원본 `values.yaml` 파일은 매우 크기 때문에, 변경할 부분만 별도의 파일에 작성하여 적용하는 것이 효율적입니다.

## 📝 주요 변경 사항 (Key Changes)

이 프로젝트에서 적용된 주요 설정은 다음과 같습니다:

1.  **NodeSelector 설정**: 모니터링 스택의 주요 구성 요소가 **마스터 노드**에 설치되도록 고정.
2.  **Grafana 새로고침 주기**: 대시보드의 최소 새로고침 주기를 5초에서 **1초**로 단축.
3.  **Scrape Config 추가**: 마스터 노드의 `nvidia-exporter` 메트릭을 수집하도록 설정.
4.  **수집 주기(Interval) 단축**: Prometheus 및 Node Exporter의 수집 주기를 **1초**로 설정하여 실시간성을 강화.

## 📄 values.yaml 예시

사용할 `values.yaml` 파일의 내용은 [values.yaml](values.yaml)을 참고하세요.

> **팁**: 현재 적용된 설정을 추출하려면 다음 명령어를 사용하세요.
> ```bash
> helm get values prometheus --namespace monitoring > current-values.yaml
> ```

## ⚙️ 설정 적용 방법

작성한 `values.yaml` 파일을 적용하여 릴리즈를 업그레이드합니다.

```bash
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f values.yaml
```
