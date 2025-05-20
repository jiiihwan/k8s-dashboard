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
