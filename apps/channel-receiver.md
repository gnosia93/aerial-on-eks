## 채널 리시버 (RX) ##

NVIDIA PyAerial은 내부적으로 NVIDIA cuPHY 가속 라이브러리를 호출하므로, 일반적인 Python 연산보다 수십 배 빠른 GPU 가속 신호 처리를 수행한다.
이 샘플은 gRPC로 들어온 IQ 데이터를 받아 채널 추정 및 등화(Equalization)를 수행한다.

### cuPHY 의 이해 ###
cuPHY는 NVIDIA의 Aerial SDK에 포함된 핵심 라이브러리로, 5G NR(New Radio) 물리 계층(L1, PHY)의 신호 처리 기능을 GPU에서 가속화하기 위해 설계되었다. 
* 인라인 가속(Inline Acceleration): 별도의 하드웨어 가속기 없이 고성능 GPU 메모리 내에서 물리 계층 데이터 전체를 처리하여 지연 시간을 최소화하고 효율성을 극대화
* O-RAN 준수: O-RAN 7.2x 프런트홀 분할(split) 옵션을 지원하며, 3GPP Release 15 사양을 준수
* 통합 파이프라인: 채널 추정, 등화(Equalization), 빔포밍, LDPC 디코딩 등 계산 집약적인 작업을 병렬로 처리

### PyAerial 수신부 알고리즘 (pyaerial_rx.py) ###
* 일반 환경에서 import nvidia.aerial을 사용하려면 사전에 SDK 라이선스 승인이 필요할 수 있다.
```
import grpc
import signal_pb2
import signal_pb2_grpc
import numpy as np
import tensorflow as tf
from nvidia.aerial import pyaerial as pa 

class PyAerialReceiver(signal_pb2_grpc.SignalStreamerServicer):
    def __init__(self):
        self.context = pa.Context()
        self.channel_estimator = pa.l1.ChannelEstimator(self.context)

    def StreamIQ(self, request_iterator, context):
        for data in request_iterator:
            iq_samples = np.frombuffer(data.samples, dtype=np.complex64)
            iq_tensor = tf.convert_to_tensor(iq_samples.reshape(data.batch_size, -1))

            # 2. PyAerial 알고리즘 실행 (GPU 가속 핵심 구간)
            # 여기서는 예시로 수신 신호의 위상을 보정하거나 복조를 준비합니다.
            processed_data = self.run_pyaerial_pipeline(iq_tensor)
            
            print(f"처리 완료: {len(processed_data)} 심볼 해독 중...")
        return signal_pb2.Empty()

    def run_pyaerial_pipeline(self, iq_tensor):
        """
        NVIDIA cuPHY 라이브러리를 활용한 물리 계층 처리
        """
        # 실제 PyAerial API를 사용하여 FFT, 채널 추정, 복조 등을 수행
        return iq_tensor * 1.0          # 가상의 처리 로직 / 파이프라인 정의 필요


def serve():
    server = grpc.server(tf.distribute.cluster_resolver.SimpleClusterResolver().num_accelerators() > 0 
                         and grpc.thread_pool_executor(max_workers=10))
    signal_pb2_grpc.add_SignalStreamerServicer_to_server(PyAerialReceiver(), server)
    server.add_insecure_port('[::]:50051')
    server.start()
    server.wait_for_termination()

if __name__ == "__main__":
    serve()
```

### Dockerfile ###
```
FROM nvcr.io/nvidia/cuda:12.2.0-base-ubuntu22.04

# 필수 패키지 및 파이썬 설치
RUN apt-get update && apt-get install -y python3-pip python3-dev && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /app

# 의존성 파일 복사 및 설치
# nvidia-aerial-pyaerial 등은 NVIDIA 전용 리포지토리 설정이 필요할 수 있음
COPY requirements.txt .
RUN pip3 install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 50051
ENV NVIDIA_VISIBLE_DEVICES all
ENV NVIDIA_DRIVER_CAPABILITIES compute,utility

CMD ["python3", "main.py"]
```

### ecr 푸시 ###
```
#!/bin/bash

# 변수 설정
REGION="ap-northeast-2"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REPO_NAME="cuphy-service"
TAG="latest"
ECR_URL="${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"

# 1. ECR 로그인
aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ECR_URL}

# 2. 리포지토리가 없으면 생성
aws ecr describe-repositories --repository-names ${REPO_NAME} || \
aws ecr create-repository --repository-name ${REPO_NAME}

# 3. 이미지 빌드 (NVIDIA 런타임 고려)
docker build -t ${REPO_NAME} .

# 4. 태그 생성 및 푸시
docker tag ${REPO_NAME}:${TAG} ${ECR_URL}/${REPO_NAME}:${TAG}
docker push ${ECR_URL}/${REPO_NAME}:${TAG}

echo "푸시 완료: ${ECR_URL}/${REPO_NAME}:${TAG}"
```

### Pod 샐행하기 ###
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cuphy-receiver
  labels:
    app: cuphy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cuphy
  template:
    metadata:
      labels:
        app: cuphy
    spec:
      # L1 처리를 위한 저지연 통신이 필요한 경우 hostNetwork 사용 고려
      hostNetwork: true 
      containers:
      - name: aerial-container
        image: ${ECR_URL}/${REPO_NAME}:latest
        ports:
        - containerPort: 50051
        resources:
          limits:
            nvidia.com/gpu: 1             # GPU 1개 할당
        env:
        - name: NVIDIA_DRIVER_CAPABILITIES
          value: "compute,utility,video"
---
apiVersion: v1
kind: Service
metadata:
  name: cuphy-grpc-svc
spec:
  clusterIP: None             # 헤드리스 설정
  selector:
    app: cuphy
  ports:
  - port: 50051
    targetPort: 50051
```

