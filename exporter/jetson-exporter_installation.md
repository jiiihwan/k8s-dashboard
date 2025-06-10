# 🛠️ jetson stats exporter설치
- based on https://github.com/laminair/jetson_stats_node_exporter
- linux service가 아닌 k8s의 pod로 띄울 수 있게 변형했다

## 🔨 0. jetson-exporter 바로 설치
직접 제작하는 방법을 따라하고 싶다면 Dockerfile 작성부터 따라하기
그렇지 않다면 아래에 있는 과정만 하면 된다

### git clone
```bash
git clone https://github.com/jiiihwan/k8s-dashboard/tree/main/exporter
```

### 모두 적용
```bash
kubectl apply -f jetson-exporter-daemonset.yaml
kubectl apply -f jetson-exporter-service.yaml -n monitoring
kubectl apply -f jetson-exporter-servicemonitor.yaml -n monitoring
```


## 📄 1. Dockerfile 작성
```bash
vim Dockerfile
```

[Dockerfile](Dockerfile) 참고

## 🔨 2. nerdctl 및 buildkit 설치
이미지 build를 위해 보통 docker를 사용하지만 컨테이너 런타임을 containerd로 사용하고 있으므로 nerdctl과 buildkit을 사용한다

### nerdctl 파일용 폴더 생성
```bash
mkdir nerdctl
cd nerdctl
```

### nerdctl 설치
```bash
curl -s https://api.github.com/repos/containerd/nerdctl/releases/latest \
| grep "browser_download_url.*linux-arm64.tar.gz" \
| cut -d '"' -f 4 \
| wget -i -
```
### 압축해제
```
tar xzvf nerdctl-full-2.0.4-linux-arm64.tar.gz
```

### buildkit 포함 nerdctl 설치
```bash
sudo cp bin/nerdctl /usr/local/bin/
sudo cp bin/buildctl /usr/local/bin/
sudo cp bin/buildkitd /usr/local/bin/
```
### 버전 확인
```
nerdctl --version
```

## 🐋 3. 이미지 build & push

### buildkitd 실행
```
sudo nohup buildkitd > /dev/null 2>&1 &
```

### l4t basefile 을 위해서 ngc회원가입 및 로그인
api키 발급(https://org.ngc.nvidia.com/setup/api-keys)
```bash
nerdctl login nvcr.io
Enter Username: $oauthtoken
Enter Password: <APIKEY>
```

### dockerfile 빌드
직접 빌드를 한다면 build 명령어의 본인의 도커허브 레포지토리를 쓰면 된다.

```bash
cd ~/jetson_stats_node_exporter
nerdctl build -t yjh2353693/jetson-exporter:latest .
```
### Dockerhub에 푸시

Dockerhub 회원가입 필요
```
nerdctl push yjh2353693/jetson-exporter:latest
```

## 📤 4. Daemonset 작성 및 배포
- 마스터노드에서 작성
- 포트는 metrics-server가 기본적으로 9100포트를 사용하고 있으므로 9101포트를 사용하도록 한다

```bash
vim jetson-exporter-daemonset.yaml
```

[jetson-exporter-daemonset.yaml](https://github.com/jiiihwan/k8s-dashboard/blob/main/exporter/jetson-exporter-daemonset.yaml) 참고

```bash
kubectl apply -f jetson-exporter-daemonset.yaml
kubectl get pods -n monitoring -o wide

#restart 할때
kubectl rollout restart daemonset jetson-exporter -n monitoring
```

## 🖥️ 5. 서비스 & 서비스모니터 설정
```bash
vim jetson-exporter-service.yaml
```

[jetson-exporter-service.yaml](https://github.com/jiiihwan/k8s-dashboard/blob/main/exporter/jetson-exporter-service.yaml) 참고

```bash
vim jetson-exporter-servicemonitor.yaml
```

[jetson-exporter-servicemonitor.yaml](https://github.com/jiiihwan/k8s-dashboard/blob/main/exporter/jetson-exporter-servicemonitor.yaml) 참고

```bash
kubectl apply -f jetson-exporter-service.yaml -n monitoring
kubectl apply -f jetson-exporter-servicemonitor.yaml -n monitoring
```

<details>
<summary> <strong> <h2> 📌[동작과정] </strong> </summary>

### ✅**1단계: Docker 이미지 준비**

### 🎯 목표: Jetson에서 동작할 수 있는 exporter 컨테이너 환경 만들기

- `jetson_stats_node_exporter` 소스를 기반으로 `l4t-base:r36.2.0` 이미지를 사용해 Dockerfile 작성
- 필요한 Python 라이브러리 (`jetson_stats`, `prometheus_client`, `schedule` 등) 설치
- Docker 이미지 빌드
- DockerHub로 푸시

### ✅ **2단계: DaemonSet으로 Pod 자동 배포**

### 🎯 목표: Jetson Orin Nano 노드마다 exporter Pod를 자동 실행

- `DaemonSet`은 Jetson 노드(예: `arm64`)마다 1개의 Pod을 배포
- 전 단계에서 만든 Docker 이미지를 기반으로 Pod 내부에서는  실행
    - `Pod`에는 라벨이 붙음: app: jetson-exporter
- Pod의 내부 포트 `9101`에서 `/metrics` 엔드포인트 열림

### ✅ **3단계: Service로 Pod 묶기**

### 🎯 목표: Prometheus가 exporter Pod에 고정된 경로로 접근 가능하도록 함

- `Service`는 Pod에 붙은 라벨 `app=jetson-exporter`를 selector로 설정
    - 내부적으로 Pod IP가 바뀌어도 항상 같은 Service 주소로 접근 가능
    - 포트 이름은 `metrics`, 포트는 `9101`로 지정

### ✅ **4단계: ServiceMonitor 생성**

### 🎯 목표: Prometheus가 위 Service를 자동으로 감지하고 scrape 하도록 연결

- Prometheus Operator는 설치 시 CRD로 `ServiceMonitor` 리소스를 생성할 수 있게 해줌
- `ServiceMonitor`는 Service를 라벨로 찾아서 연결
    - scrape할 포트(`metrics`)와 주기(`1s`)도 정의

### ✅ **5단계: Prometheus Operator가 감지**

### 🎯 내부 동작 순서:

1. Prometheus Operator는 ServiceMonitor를 **주기적으로 감시**
2. `release: prometheus` 라벨이 붙은 ServiceMonitor만 인식
3. 이걸 기반으로 Prometheus의 scrape 설정을 **자동 업데이트**함 (`scrape_configs`에 job 추가됨)

### ✅ **6단계: Prometheus가 실제로 수집 시작**

### 🎯 목표: exporter에서 메트릭을 받아와 저장

- Prometheus는 Service를 통해 Pod의 `/metrics` 엔드포인트에 접근
- exporter는 `jetson_gpu_temp_c`, `jetson_power_usage_watts` 같은 메트릭을 내보냄
    - Prometheus는 이 데이터를 수집하고 시계열 DB에 저장
