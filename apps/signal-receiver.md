## 채널 리시버 (RX) ##

NVIDIA PyAerial은 내부적으로 NVIDIA cuPHY 가속 라이브러리를 호출하므로, 일반적인 Python 연산보다 수십 배 빠른 GPU 가속 신호 처리를 수행한다.
이 샘플은 gRPC로 들어온 IQ 데이터를 받아 채널 추정 및 등화(Equalization)를 수행한다.

#### cuPHY 의 이해 ####
cuPHY는 NVIDIA의 Aerial SDK에 포함된 핵심 라이브러리로, 5G NR(New Radio) 물리 계층(L1, PHY)의 신호 처리 기능을 GPU에서 가속화하기 위해 설계되었다. 
* 인라인 가속(Inline Acceleration): 별도의 하드웨어 가속기 없이 고성능 GPU 메모리 내에서 물리 계층 데이터 전체를 처리하여 지연 시간을 최소화하고 효율성을 극대화
* O-RAN 준수: O-RAN 7.2x 프런트홀 분할(split) 옵션을 지원하며, 3GPP Release 15 사양을 준수
* 통합 파이프라인: 채널 추정, 등화(Equalization), 빔포밍, LDPC 디코딩 등 계산 집약적인 작업을 병렬로 처리

#### 1. PyAerial 수신부 알고리즘 (pyaerial_rx.py) ####
```
import grpc
import signal_pb2
import signal_pb2_grpc
import numpy as np
import tensorflow as tf
from nvidia.aerial import pyaerial as pa # NVIDIA PyAerial 라이브러리

class PyAerialReceiver(signal_pb2_grpc.SignalStreamerServicer):
    def __init__(self):
        # 1. PyAerial GPU 가속 컨텍스트 초기화
        self.context = pa.Context()
        # 2. 초고속 신호 처리를 위한 L1 처리 객체 생성 (예: 채널 추정기)
        self.channel_estimator = pa.l1.ChannelEstimator(self.context)
        print("PyAerial GPU 가속 수신기가 준비되었습니다.")

    def StreamIQ(self, request_iterator, context):
        for data in request_iterator:
            # 1. 바이너리 데이터를 넘파이 배열로 복원 (Complex64)
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
        # 실제 PyAerial API를 사용하여 FFT, 채널 추정, 복조 등을 수행합니다.
        # 자세한 API는 NVIDIA Aerial SDK 문서를 참조하십시오.
        return iq_tensor * 1.0 # (가상의 처리 로직)

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

#### 2. 배포 시 핵심 사항 ####
* NVIDIA 커널 드라이버: 이 코드가 돌아가려면 호스트 머신에 NVIDIA Aerial SDK 전용 드라이버와 CUDA가 올바르게 설치되어 있어야 합니다.
* 패키지 가용성: nvidia.com 라이브러리는 NVIDIA NGC 컨테이너 환경에서 가장 잘 작동합니다. 일반 환경에서 import nvidia.aerial을 사용하려면 사전에 SDK 라이선스 승인이 필요할 수 있습니다.

#### 3. k8s에서의 흐름 요약 ####
* Sionna Pod: 가상의 무선 채널 데이터를 생성하여 gRPC로 전송.
* PyAerial Pod: gRPC로 받은 RAW 데이터를 NVIDIA GPU를 통해 실시간 해독.
* 결과: 복구된 비트 데이터를 상위 애플리케이션으로 전달.


---

Shared Memory(공유 메모리) 방식은 네트워크 스택을 거치지 않고 메모리 주소를 직접 공유하기 때문에 gRPC보다 훨씬 빠릅니다.
Python에서 이를 구현하려면 multiprocessing.shared_memory를 사용하거나, 더 고성능을 위해 NVIDIA cuMem (CUDA Inter-process Communication)을 활용하지만, 여기서는 가장 일반적이고 구현이 쉬운 POSIX Shared Memory 기반의 샘플을 보여드립니다.

#### 1. [Sionna Pod] 신호 생성 및 메모리 쓰기 ####
생성한 IQ 샘플을 공유 메모리 영역에 쓰고, Event나 Semaphore 역할을 할 플래그를 업데이트합니다.
```
import numpy as np
from multiprocessing import shared_memory
import sionna as sn
import time

# 1. 공유 메모리 생성 (약 10MB 크기: Complex64 125만 샘플 분량)
shm = shared_memory.SharedMemory(name="iq_stream", create=True, size=10 * 1024 * 1024)

# 2. 신호 생성기 설정 (Sionna)
binary_source = sn.utils.BinarySource()
qam_source = sn.utils.QAMSource(num_bits_per_symbol=4)

try:
    print("Sionna: 공유 메모리에 신호를 쓰고 있습니다...")
    while True:
        # 신호 생성
        bits = binary_source([1, 1024])
        x = qam_source(bits).numpy() # (1, 256) 형태의 complex64
        
        # 3. 공유 메모리에 데이터 복사
        # 앞부분 4바이트는 데이터 크기나 시퀀스 번호로 활용 가능
        shared_array = np.ndarray(x.shape, dtype=x.dtype, buffer=shm.buf)
        shared_array[:] = x[:]
        
        time.sleep(0.001) # 실시간 주기 조절 (1ms)
except KeyboardInterrupt:
    shm.close()
    shm.unlink()
```

#### 2. [PyAerial Pod] 메모리 읽기 및 알고리즘 실행 ####
동일한 name으로 메모리에 접근하여 데이터를 즉시 가져옵니다.
```
import numpy as np
from multiprocessing import shared_memory
from nvidia.aerial import pyaerial as pa
import time

# 1. 기존 공유 메모리에 연결
shm = shared_memory.SharedMemory(name="iq_stream")

# 2. PyAerial 초기화
ctx = pa.Context()
# 예: L1 처리를 위한 도구 세팅
# estimator = pa.l1.ChannelEstimator(ctx)

try:
    print("PyAerial: 공유 메모리에서 데이터를 읽어 GPU로 처리합니다...")
    while True:
        # 3. 메모리에서 직접 numpy 배열로 매핑 (복사 없음)
        # 생성단과 동일한 shape와 dtype 지정
        iq_data = np.ndarray((1, 256), dtype=np.complex64, buffer=shm.buf)
        
        if iq_data.any():
            # 4. PyAerial 알고리즘 실행
            # processed = estimator.run(iq_data)
            print(f"처리 중... 첫 번째 샘플: {iq_data[0][0]}")
            
        time.sleep(0.001)
except KeyboardInterrupt:
    shm.close()
```

#### 3. K8s 설정 (Shared Memory 활성화) ####
두 컨테이너가 /dev/shm을 통해 동일한 자원을 보게 하려면 emptyDir 설정이 필수입니다.
```
apiVersion: v1
kind: Pod
metadata:
  name: aerial-shm-pod
spec:
  # 중요: 두 컨테이너가 동일한 IPC 네임스페이스를 공유하도록 설정
  hostIPC: true 
  containers:
  - name: sionna-gen
    image: my-sionna-gen:latest
    volumeMounts:
    - mountPath: /dev/shm
      name: cache-volume
  - name: pyaerial-rx
    image: nvidia-pyaerial:latest
    volumeMounts:
    - mountPath: /dev/shm
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir:
      medium: Memory # RAM을 사용하여 디스크 I/O 병목 제거
```

#### 핵심 장점 ####
* Zero-Copy: 데이터가 네트워크 카드나 커널을 거치지 않고 메모리 주소만 넘겨주므로 지연 시간이 거의 없습니다.
* 간결함: 복잡한 gRPC 프로토콜 정의 없이 넘파이 배열 구조만 맞추면 됩니다.
* 주의: 위 코드는 데이터가 써지는 동안 읽는 경합 현상(Race Condition)이 발생할 수 있습니다. 실제 구현 시에는 Python 주식/거래 시스템처럼 세마포어나 간단한 플래그 변수를 메모리 첫 칸에 두어 쓰기 완료 여부를 체크하는 것이 좋습니다.


