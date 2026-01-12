Sionna를 이용해 실시간으로 IQ 샘플(신호)을 생성하고, 이를 다른 Pod(PyAerial 등)로 전송하기 위한 기초 Python 코드 예시입니다.
이 코드는 송신단(Transmitter) + 채널(Channel) 역할을 하며, 생성된 데이터를 바이너리 형태로 출력하거나 공유 메모리/네트워크로 쏠 준비를 하는 구조입니다.

#### 1. Sionna 기반 신호 생성기 (signal_gen.py) ####
```
import tensorflow as tf
import sionna as sn
from sionna.utils import BinarySource, QAMSource, Transmitter
from sionna.channel import AWGN

class SignalGenerator(tf.keras.Model):
    def __init__(self, num_bits_per_symbol=4): # 16QAM
        super().__init__()
        self.binary_source = BinarySource()
        self.qam_source = QAMSource(num_bits_per_symbol)
        self.channel = AWGN()

    def generate_batch(self, batch_size, ebno_db=20.0):
        # 1. 랜덤 비트 생성
        bits = self.binary_source([batch_size, 1024]) 
        
        # 2. QAM 변조 (신호 생성)
        x = self.qam_source(bits)
        
        # 3. 무선 채널 통과 (노이즈 추가)
        y = self.channel([x, ebno_db])
        
        # y는 complex64 형태의 IQ 샘플 배열입니다.
        return y

# 실행부
gen = SignalGenerator()

print("--- 실시간 신호 생성 시작 ---")
while True:
    # 실시간으로 64개 배치의 IQ 샘플 생성
    iq_samples = gen.generate_batch(batch_size=64)
    
    # 여기서 iq_samples.numpy()를 바이너리로 변환하여 
    # Shared Memory나 gRPC로 PyAerial Pod에 던집니다.
    print(f"Generated {iq_samples.shape} IQ samples (Complex64)")
```    


#### 2. 쿠버네티스 배포를 위한 Dockerfile (NVIDIA 공식 이미지 활용) ####
Sionna는 GPU 가속을 위해 NVIDIA TensorFlow 컨테이너 위에서 돌리는 것이 가장 안정적입니다.
```
FROM nvcr.io/nvidia/tensorflow:23.10-tf2-py3

# Sionna 및 필요한 라이브러리 설치
RUN pip install sionna

# 소스 복사
COPY signal_gen.py /app/signal_gen.py
WORKDIR /app

# GPU를 사용해 실행
CMD ["python", "signal_gen.py"]
코드를 사용할 때는 주의가 필요합니다.
```

#### 3. K8s 아키텍처 구현 시 주의사항 ####
* GPU 할당: 이 Pod은 신호를 실시간으로 "계산"해서 만들어내므로 NVIDIA GPU Operator를 통해 nvidia.com: 1 리소스 할당이 필요합니다.
데이터 전달: 생성된 iq_samples를 PyAerial Pod로 보낼 때, 성능 손실을 줄이려면 gRPC 스트리밍을 사용해 데이터를 직렬화하여 쏘는 루틴을 위 while 문 안에 추가해야 합니다.
이 신호 생성기 코드에 gRPC 전송 로직까지 포함된 전체 샘플이 필요하신가요? 아니면 Shared Memory 설정용 YAML이 더 궁금하신가요?

## gRPC ##
실시간 신호 처리 환경을 위한 gRPC 기반 송신 코드와 Pod 간 초고속 데이터 공유를 위한 Shared Memory(POSIX) 설정 YAML 샘플입니다.
1. gRPC 로직이 추가된 [Sionna Pod] 코드
이 코드는 신호를 생성하여 다른 Pod(PyAerial)로 끊임없이 스트리밍합니다.
```
proto 정의 (signal.proto)
proto
syntax = "proto3";

service SignalStreamer {
  rpc StreamIQ (stream IQData) returns (Empty) {}
}

message IQData {
  bytes samples = 1; // 변환된 complex64 바이너리 데이터
  int32 batch_size = 2;
}

message Empty {}
```

Python 송신부 (signal_gen_grpc.py)
```
import grpc
import signal_pb2, signal_pb2_grpc
import sionna as sn
import tensorflow as tf

class SignalGenStreamer:
    def __init__(self, target_address='pyaerial-service:50051'):
        self.channel = grpc.insecure_channel(target_address)
        self.stub = signal_pb2_grpc.SignalStreamerStub(self.channel)
        # Sionna 설정
        self.binary_source = sn.utils.BinarySource()
        self.qam_source = sn.utils.QAMSource(num_bits_per_symbol=4) # 16QAM

    def generate_and_stream(self):
        def request_generator():
            while True:
                # 1. 신호 생성 (Sionna)
                bits = self.binary_source([64, 1024])
                x = self.qam_source(bits)
                
                # 2. 바이너리로 직렬화 (Complex64 -> Bytes)
                iq_bytes = x.numpy().tobytes()
                
                # 3. gRPC 메시지로 포장
                yield signal_pb2.IQData(samples=iq_bytes, batch_size=64)
        
        # 스트리밍 시작
        self.stub.StreamIQ(request_generator())

if __name__ == "__main__":
    streamer = SignalGenStreamer()
    streamer.generate_and_stream()
```

## 2. Shared Memory 설정용 K8s YAML ##
gRPC보다 더 빠른 데이터 전달이 필요할 때 사용하는 방식입니다. /dev/shm을 공유하여 메모리 복사 없이 데이터를 주고받습니다.
Kubernetes Shared Memory 가이드에 따라 emptyDir의 매체를 Memory로 설정합니다.

```
apiVersion: v1
kind: Pod
metadata:
  name: aerial-testbed
spec:
  containers:
  - name: sionna-gen
    image: my-sionna-gen:latest
    resources:
      limits:
        nvidia.com: 1  # GPU 가속 필요
    volumeMounts:
    - mountPath: /dev/shm
      name: dshm
  - name: pyaerial-rx
    image: nvidia-pyaerial:latest
    resources:
      limits:
        nvidia.com: 1
    volumeMounts:
    - mountPath: /dev/shm
      name: dshm
  volumes:
  - name: dshm
    emptyDir:
      medium: Memory
      sizeLimit: "2Gi" # 생성할 신호 데이터 크기에 맞게 조절
```
아키텍처 구현 핵심 포인트
* gRPC 방식: Pod가 서로 다른 노드에 있어도 작동하므로 확장성이 좋습니다.
* Shared Memory 방식: 한 노드(EC2 G5 등) 안에서만 작동하지만, 지연 시간이 거의 없습니다. 실시간 L1 처리에 강력히 추천합니다.
* Docker 공유: Shared Memory를 쓰려면 두 컨테이너가 동일한 IPC 네임스페이스를 공유해야 할 수도 있으므로, 필요시 hostIPC: true 설정을 쿠버네티스 공식 문서를 참조하여 추가하세요.


----
네, 작성하신 코드는 CPU에서도 문제없이 돌아갑니다.
NVIDIA Sionna 공식 문서에 따르면, 시오나는 TensorFlow를 기반으로 작동하기 때문에 별도의 GPU가 없으면 자동으로 CPU를 사용하여 연산을 수행합니다.
다만, 몇 가지 참고하실 점이 있습니다:
속도 차이: 시오나는 대규모 병렬 연산(Monte-Carlo 시뮬레이션 등)에 최적화되어 있어, NVIDIA 개발자 블로그에서 강조하듯 GPU(CUDA) 사용 시 수십~수백 배 더 빠릅니다. 실시간으로 PyAerial에 던질 만큼 많은 양을 생성하신다면 CPU 점유율이 꽤 높을 거예요.
데이터 타입: 코드의 iq_samples는 complex64인데, 이를 바이너리로 변환하여 전송할 때 엔디안(Endian)이나 메모리 레이아웃이 PyAerial 측 규격(보통 int16 I/Q로 변환이 필요한 경우도 있음)과 맞는지 확인이 필요합니다.
최적화: CPU 환경이라면 batch_size를 적절히 조절하여 지연 시간(Latency)을 체크해 보시는 것이 좋습니다.
PyAerial Pod로 데이터를 넘기실 때, gRPC를 쓰실 건가요 아니면 Shared Memory 방식을 쓰실 건가요? 방식에 따라 넘파이(Numpy) 직렬화 팁을 드릴 수 있습니다.

