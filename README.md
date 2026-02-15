# 💻 k8s-dashboard

[**English**](README.en.md) | [**한국어**](README.md)

> ### k8s dashboard with real-time resource utilization

기본 Kubernetes 대시보드에서 지원하지 않는 GPU 사용량을 포함하여, 노드의 실시간 리소스 사용량을 모니터링하기 위해 **Prometheus**와 **Grafana**를 사용하여 구현한 커스텀 대시보드입니다.

## 🌟 주요 기능 및 지원 리소스

다음 리소스에 대한 실시간 모니터링을 지원합니다:

1.  **CPU** 사용량 (%)
    - 코어별 사용량 포함
2.  **GPU** 사용량 (%)
3.  **RAM** 사용량 (GB & %)
4.  **Network** 사용량 (Bit/sec)
    - 송수신(Transmitted & Received) 데이터
5.  **NPU** 사용량 (%)

## 🧱 시스템 아키텍처

이 시스템은 Kubernetes 클러스터 상에서 동작하며, Prometheus가 각 워커 노드에 설치된 Exporter로부터 메트릭을 수집하고, 이를 Grafana를 통해 실시간으로 시각화합니다.

![System Architecture](https://github.com/user-attachments/assets/f76048d4-f741-41ec-9a56-f105da05756a)

1.  **Prometheus Operator**: Kubernetes 내에서 Prometheus 관련 모니터링 리소스를 관리하는 컨트롤러입니다. 사용자가 Prometheus, ServiceMonitor 등을 Custom Resource(CR)로 정의하면, Operator가 이를 감지하여 설정을 자동으로 업데이트합니다.
2.  **DaemonSet**: 각 워커 노드에 Exporter를 자동으로 배포하는 역할을 합니다. `device=jetson`과 같은 특정 태그가 붙은 Jetson Orin Nano 노드에는 `nodeSelector`를 통해 적절한 Exporter 파드가 배치되도록 보장합니다. 노드가 재시작되거나 파드가 삭제되어도 DaemonSet이 자동으로 복구합니다. 각 워커 노드에서는 일반적인 시스템 메트릭을 수집하는 node-exporter와, GPU 사용량 등 Jetson 특화 메트릭을 수집하는 jetson-exporter가 함께 실행됩니다.
3.  **ServiceMonitor**: Prometheus Operator가 관리하는 Custom Resource로, Prometheus가 Exporter로부터 메트릭을 자동으로 검색(Discover)하고 수집(Scrape)할 수 있게 합니다. 각 Exporter는 유동적인 IP를 가진 파드에서 실행되므로, 고정된 ClusterIP로 접근할 수 있도록 Kubernetes Service를 생성합니다. 이 Service는 `app=jetson-exporter`와 같은 라벨 셀렉터를 사용해 올바른 파드를 가리킵니다. Prometheus Operator는 ServiceMonitor를 지속적으로 감시하며, 특정 라벨과 일치하는 Service를 찾아 해당 Exporter의 `/metrics` 엔드포인트를 수집하도록 Prometheus를 설정합니다. ServiceMonitor는 Operator가 인식할 수 있도록 `release=prometheus`와 같은 라벨을 포함해야 합니다.
4.  **Exporters**: 각 Exporter는 `/metrics` HTTP 엔드포인트에서 시스템 메트릭을 Prometheus 호환 형식으로 노출합니다. Prometheus는 pull 방식을 사용하여 주기적으로 이 엔드포인트를 scrape하여 모든 시계열 데이터를 내부 TSDB(Time Series Database)에 저장합니다.
5.  **Grafana**: Prometheus가 수집한 데이터를 PromQL 쿼리를 사용하여 시각화합니다. 사용자는 각 노드 및 전체 클러스터의 CPU, GPU, 메모리, 네트워크 사용량 등을 실시간으로 모니터링하여 리소스 활용 상태를 직관적으로 파악할 수 있습니다.

## ⚙️ 하드웨어 및 환경 구성

| 노드 유형 | 장비 | OS |
| :--- | :--- | :--- |
| **Master Node** | Desktop | Ubuntu 22.04 |
| **Worker Node** | 2 × NVIDIA Jetson Orin Nano | JetPack 6.0 |
| **Worker Node** | 1 × Raspberry Pi 5 with Hailo | Raspberry Pi OS (64-bit) |

## 🛠️ 설치 및 시작하기

이 프로젝트의 핵심인 Prometheus와 Grafana 대시보드 구축을 위한 단계입니다:

-   **Step 1: Prometheus & Grafana**
    -   [📂 Prometheus & Grafana](https://github.com/jiiihwan/k8s-dashboard/tree/main/Prometheus%26Grafana)
-   **Step 2: Exporters** (GPU/NPU 수집기)
    -   [📂 Nvidia-exporter](https://github.com/jiiihwan/k8s-dashboard/tree/main/nvidia-exporter) (Desktop GPU용)
    -   [🔗 Jetson_exporter](https://github.com/jiiihwan/jetson_exporter) (Orin Nano용)
    -   [🔗 Hailo_exporter](https://github.com/jiiihwan/hailo_exporter) (RPi 5 NPU용)

### 📚 참고 자료 (References)

Kubernetes 클러스터 설정이나 기본 대시보드 구성이 필요한 경우 아래 가이드를 참고하세요:

-   [📂 Kubernetes 설정 가이드](https://github.com/jiiihwan/k8s-dashboard/tree/main/kubernetes)
-   [📂 K8s Dashboard 구성 가이드](https://github.com/jiiihwan/k8s-dashboard/tree/main/kubernetes/k8s_dashboard)

## 🔋 대시보드 미리보기

![Grafana Dashboard](https://github.com/user-attachments/assets/f8c5a38a-8382-4edc-b511-a6b56bd2e01a)

-   **상단**: CPU, GPU, RAM, Network 순서
-   **좌측 열**: 전체 클러스터, 마스터 노드, 워커 노드 순
-   **우측 열**: Orin Nano의 코어별 CPU 사용량
-   **차트**: 노드별 네트워크 사용량 비율

**JSON 템플릿:**
-   [K8s Cluster Dashboard.json](https://github.com/jiiihwan/k8s-dashboard/blob/main/Prometheus%26Grafana/K8s%20Cluster%20Dashboard.json)
-   [K8s Cluster Dashboard with RPI.json](https://github.com/jiiihwan/k8s-dashboard/blob/main/Prometheus%26Grafana/K8s%20Cluster%20Dashboard%20with%20RPI.json)
