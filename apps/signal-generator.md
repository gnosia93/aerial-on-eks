## 시그널 제너레이터 (TX) ##
NVIDIA PyAerial 및 Triton gRPC 인프라 환경에서 Sionna 생성 신호를 실시간 전송하기 위한 gRPC 구현 코드이다. TensorFlow complex64 텐서를 tobytes()로 직렬화하여 protobuf bytes 필드에 담아 보내는 방식이 가장 효율적이다.

### 1. Proto 정의 (signal.proto) ###
grpc 패키지를 설치한다. 
```
pip install grpcio grpcio-tools
```
먼저 IQ 샘플 데이터를 주고받기 위한 메시지 형식을 정의한다.

[signal.proto]
```
syntax = "proto3";

service SignalStreamer {
  rpc SendSignal(SignalRequest) returns (SignalResponse);
}

message SignalRequest {
  bytes iq_data = 1;                // complex64 바이너리 데이터
  repeated int32 shape = 2;         // [batch, samples]
}

message SignalResponse {
  bool success = 1;
}
```
파이썬 코드로 변환 (컴파일) 한다.
```
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. signal.proto
```
[결과]
```
signal_pb2.py: 메시지 규격(SignalRequest, SignalResponse)이 정의된 파일.
signal_pb2_grpc.py: 서비스 서버/클라이언트 로직(SignalStreamer)이 정의된 파일.
```

### 2. SignalGenerator ###
gRPC-Python 스트리밍 라이브러리를 활용하여 SignalGenerator와 연결한다.
```
import os
import grpc
import signal_pb2
import signal_pb2_grpc
import tensorflow as tf
import sionna as sn
from sionna.utils import BinarySource, QAMSource
from sionna.channel import AWGN

# reciever_channel = "pyaerial-service.default.svc.cluster.local:50051"
# 동일 네임스페이스인 경우 아래와 같이 생략가능.
reciever_channel = os.environ.get("RECEIVER_ADDRESS", "pyaerial-service:50051")

class SignalGenerator(tf.keras.Model):
    def __init__(self, num_bits_per_symbol=4):
        super().__init__()
        self.binary_source = BinarySource()                 # 비트 생성기 (0과 1로 구성된 랜덤 비트를 생성)
        self.qam_source = QAMSource(num_bits_per_symbol)    # 디지털 변조기 (4비트를 묶어 복소수 좌표계(IQ Plane)로 옮김. 4비트이므로 16-QAM 변조 수행    
        self.channel = AWGN()                               # AWGN(Additive White Gaussian Noise) 채널은 가장 기본적인 무선 통신 채널

    def generate_batch(self, batch_size, ebno_db=20.0):
        bits = self.binary_source([batch_size, 1024])       # batch_size개의 사용자(또는 스트림)에 대해 각각 1024비트씩 생성
        x = self.qam_source(bits)                           # 0/1 데이터가 실제 전파에 실릴 수 있는 복소수(Complex64) 심볼로 변환   
        y = self.channel([x, ebno_db])                      # 생성된 신호 x에 지정된 에너지 대비 노이즈 비율(ebno_db)만큼의 백색 잡음 추가  
        return y

# gRPC 클라이언트 설정
channel = grpc.insecure_channel(reciever_channel)
stub = signal_pb2_grpc.SignalStreamerStub(channel)

gen = SignalGenerator()

print("--- gRPC 기반 실시간 신호 전송 시작 ---")
try:
    while True:
        # 1. 신호 생성
        iq_samples = gen.generate_batch(batch_size=64)
        
        # 2. 직렬화 (Tensor -> Numpy -> Bytes)
        # complex64는 float32 2개(실수, 허수)로 구성됨
        data_bytes = iq_samples.numpy().tobytes()
        
        # 3. gRPC 요청 생성 및 전송
        request = signal_pb2.SignalRequest(
            iq_data=data_bytes,
            shape=list(iq_samples.shape)
        )
        
        # 동기 방식 전송 (성능을 위해 비동기 stub.SendSignal.future() 사용 권장)
        response = stub.SendSignal(request)
        
        if response.success:
            print(f"Sent {iq_samples.shape} samples via gRPC")
except KeyboardInterrupt:
    print("전송 중단")
```
* 데이터 타입: 코드의 iq_samples는 complex64인데, 이를 바이너리로 변환하여 전송할 때 엔디안(Endian)이나 메모리 레이아웃이 PyAerial 측 규격(보통 int16 I/Q로 변환이 필요한 경우도 있음)과 맞는지 확인이 필요합니다.
* PyAerial Pod 측(Server)에서는 수신한 bytes 데이터를 np.frombuffer(request.iq_data, dtype=np.complex64).reshape(request.shape)로 복원하여 cuPHY 파이프라인에 주입할 수 있습니다. 
* 수신 측에서 NVIDIA Aerial SDK의 어떤 모듈(예: PUSCH/PDSCH 디코더)로 데이터를 넘길 계획이신가요?

## Dockerfile ##
Sionna는 GPU 가속을 위해 NVIDIA TensorFlow 컨테이너 위에서 돌리는 것이 가장 좋다.
Sionna는 TensorFlow를 기반으로 작동하기 때문에 별도의 GPU가 없으면 자동으로 CPU를 사용하여 연산을 수행한다.
하지만 Sionna는 대규모 병렬 연산(Monte-Carlo 시뮬레이션 등)에 최적화되어 있어, GPU(CUDA) 사용 시 수십~수백 배 더 빠르다. 
CPU 환경이라면 batch_size를 적절히 조절하여 지연 시간(Latency)을 체크해 보시는 것이 좋다.
```
FROM nvcr.io/nvidia/tensorflow:23.10-tf2-py3

WORKDIR /app
RUN pip install sionna
COPY signal_gen.py /app/signal_gen.py

CMD ["python", "signal_gen.py"]
```

## 도커 빌드 / ecr 푸시 ##
```

export AWS_REGION=$(aws ec2 describe-availability-zones --query 'AvailabilityZones[0].RegionName' --output text)
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REPO_NAME="sionna-generator"
ECR_URL="${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
```
ecr 레포지토리를 생성하고 로그인 한다.
```
aws ecr create-repository --repository-name ${REPO_NAME} --region ${AWS_REGION}
aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_URL}
```
도커 이미지를 빌드하고 ecr 에 푸시한다.
```
docker build -t ${REPO_NAME} .
docker tag ${REPO_NAME}:latest ${ECR_URL}/${REPO_NAME}:latest

docker push ${ECR_URL}/${REPO_NAME}:latest 
```

## POD 배포하기 ##
```
apiVersion: v1
kind: Pod
metadata:
  name: sionna-generator
  labels:
    app: sionna-gen
spec:
  containers:
  - name: sionna-container
    image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com     # 이미지 주소는 본인의 ECR 주소로 수정
    
    # gRPC 연결을 위한 환경 변수 설정 (코드 내에서 os.environ으로 참조 가능)
    env:
    - name: RECEIVER_ADDRESS
      value: "pyaerial-service.default.svc.cluster.local:50051"
    
    resources:
      limits:
        nvidia.com/gpu: 1              # Sionna 가속을 위해 GPU 할당
        memory: "8Gi"
        cpu: "4"
      requests:
        nvidia.com: 1
        memory: "4Gi"
        cpu: "2"
        
    volumeMounts:                      # 대용량 신호 처리를 위한 공유 메모리 설정 (필요 시) 
    - name: dshm
      mountPath: /dev/shm
  
  volumes:
  - name: dshm
    emptyDir:
      medium: Memory
```

### 아키텍처 구현 핵심 포인트 ###
* gRPC 방식: Pod가 서로 다른 노드에 있어도 작동하므로 확장성이 좋습니다.
* Shared Memory 방식: 한 노드(EC2 G5 등) 안에서만 작동하지만, 지연 시간이 거의 없습니다. 실시간 L1 처리에 강력히 추천합니다.
* Docker 공유: Shared Memory를 쓰려면 두 컨테이너가 동일한 IPC 네임스페이스를 공유해야 할 수도 있으므로, 필요시 hostIPC: true 설정을 쿠버네티스 공식 문서를 참조하여 추가하세요.


