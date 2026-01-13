## TX ##
NVIDIA PyAerial 및 Triton gRPC 인프라 환경에서 Sionna 생성 신호를 실시간 전송하기 위한 gRPC 구현 코드이다. TensorFlow complex64 텐서를 tobytes()로 직렬화하여 protobuf bytes 필드에 담아 보내는 방식이 가장 효율적이다.

### 1. Proto 정의 (signal.proto) ###
먼저 IQ 샘플 데이터를 주고받기 위한 메시지 형식을 정의한다.

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

### 2. SignalGenerator ###
gRPC-Python 스트리밍 라이브러리를 활용하여 SignalGenerator와 연결한다.
```
import grpc
import signal_pb2
import signal_pb2_grpc
import tensorflow as tf
import sionna as sn
from sionna.utils import BinarySource, QAMSource
from sionna.channel import AWGN

class SignalGenerator(tf.keras.Model):
    def __init__(self, num_bits_per_symbol=4):
        super().__init__()
        self.binary_source = BinarySource()
        self.qam_source = QAMSource(num_bits_per_symbol)
        self.channel = AWGN()

    def generate_batch(self, batch_size, ebno_db=20.0):
        bits = self.binary_source([batch_size, 1024]) 
        x = self.qam_source(bits)
        y = self.channel([x, ebno_db])
        return y

# gRPC 클라이언트 설정
channel = grpc.insecure_channel('localhost:50051')
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

### 아키텍처 구현 핵심 포인트 ###
* gRPC 방식: Pod가 서로 다른 노드에 있어도 작동하므로 확장성이 좋습니다.
* Shared Memory 방식: 한 노드(EC2 G5 등) 안에서만 작동하지만, 지연 시간이 거의 없습니다. 실시간 L1 처리에 강력히 추천합니다.
* Docker 공유: Shared Memory를 쓰려면 두 컨테이너가 동일한 IPC 네임스페이스를 공유해야 할 수도 있으므로, 필요시 hostIPC: true 설정을 쿠버네티스 공식 문서를 참조하여 추가하세요.


