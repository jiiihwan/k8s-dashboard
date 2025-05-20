# jetson stats exporter설치
- based on https://github.com/laminair/jetson_stats_node_exporter
- linux service가 아닌 k8s의 pod로 띄울 수 있게 변형했다

## 1) Dockerfile 작성
```Dockerfile
#참고로 버전(r36.2.0)은 홈페이지에서 잘 보고 적어야함
#https://catalog.ngc.nvidia.com/orgs/nvidia/containers/l4t-base
FROM nvcr.io/nvidia/l4t-base:r36.2.0

# 컨테이너 내 작업 디렉토리 설정
WORKDIR /opt/jetson_exporter

# 필수 패키지 설치
RUN apt-get update && apt-get install -y python3-pip curl && \
    pip3 install jetson_stats prometheus_client

# requirements.txt 복사 + 설치
COPY requirements.txt .
RUN pip3 install -r requirements.txt

# jetson_stats_node_exporter 모듈 전체 복사
COPY jetson_stats_node_exporter ./jetson_stats_node_exporter

# Prometheus가 수집할 포트 열기
EXPOSE 9101

# 모듈을 실행
ENTRYPOINT ["python3", "-m", "jetson_stats_node_exporter", "--port=9101"]
```

## 2) nerdctl 및 buidkit 설치
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
### buildkitd 실행
```
sudo nohup buildkitd > /dev/null 2>&1 &
```

## 3) 이미지 build & push

### l4t basefile 을 위해서 ngc회원가입 및 로그인
- api키 발급(https://org.ngc.nvidia.com/setup/api-keys)
```bash
nerdctl login nvcr.io
Enter Username: $oauthtoken
Enter Password: <APIKEY>
```

### dockerfile 빌드
```bash
cd /home/orin1/jetson_stats_node_exporter
nerdctl build -t yjh2353693/jetson-exporter:latest .
```
### Dockerhub에 푸시
- Dockerhub 회원가입 필요
```
nerdctl push yjh2353693/jetson-exporter:latest
```

## 4) Daemonset 작성 및 배포
- 마스터노드에서 작성
- 포트는 metrics-server가 기본적으로 9100포트를 사용하고 있으므로 9101포트를 사용하도록 한다

`vim jetson-exporter-daemonset.yaml`

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: jetson-exporter
  namespace: monitoring
  labels:
    app: jetson-exporter
spec:
  selector:
    matchLabels:
      app: jetson-exporter
  template:
    metadata:
      labels:
        app: jetson-exporter
    spec:
      nodeSelector:
        kubernetes.io/arch: arm64
      containers:
        - name: jetson-exporter
          image: yjh2353693/jetson-exporter:latest #혹은 자신이 빌드한 이미지
          ports:
            - containerPort: 9101 #9101로 설정
              name: metrics
          volumeMounts:
            - name: jtop-sock
              mountPath: /run/jtop.sock
              readOnly: true
          securityContext:
            privileged: true
      volumes:
        - name: jtop-sock
          hostPath:
            path: /run/jtop.sock
            type: Socket
```
```bash
kubectl apply -f jetson-exporter-daemonset.yaml
kubectl get pods -n monitoring -o wide

#restart 할때
kubectl rollout restart daemonset jetson-exporter -n monitoring
```

## 5) 서비스 설정
`vim jetson-exporter-service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: jetson-exporter
  namespace: monitoring
  labels:
    app: jetson-exporter
spec:
  selector:
    app: jetson-exporter
  ports:
    - name: metrics
      port: 9101
      targetPort: 9101
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
