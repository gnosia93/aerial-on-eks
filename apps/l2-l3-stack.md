L2/L3 스택은 수학적 행렬 연산보다는 패킷의 순서를 맞추고(RLC), 사용자별로 자원을 할당(MAC Scheduler)하며, IP 패킷을 캡슐화하는 복잡한 논리 처리가 주를 이룹니다.
아래는 Graviton(ARM64) 노드에서 실행하기 적합한 파이썬 기반의 단순화된 L2(MAC) 스케줄러 및 데이터 처리 샘플 코드입니다. 이 코드는 TX Pod로부터 받은 데이터를 처리하여 RX Pod로 전달하는 "중계 및 제어" 역할을 수행합니다.
```
import numpy as np
import time
from concurrent.futures import ThreadPoolExecutor

class L2L3Stack:
    def __init__(self):
        # Graviton의 멀티코어를 활용하기 위한 설정
        self.executor = ThreadPoolExecutor(max_workers=8) 
        print("[L2/L3] Graviton-based Protocol Stack Initialized (ARM64)")

    def mac_scheduler(self, data):
        """
        L2 역할: 데이터를 사용자별/우선순위별로 스케줄링
        """
        # 실제로는 여기서 복잡한 Priority Queue 처리가 일어남
        processed_data = data * 1.0  # 단순 패킷 통과 시뮬레이션
        return processed_data

    def rlc_layer_processing(self, packet_id, data):
        """
        L2 역할: 패킷 번호 부여 및 재전송 제어(ARQ) 로직
        """
        # Graviton의 정수 연산 성능을 활용한 헤더 부착 및 무결성 검사
        header = f"SN-{packet_id}".encode()
        return header + data.tobytes()

    def process_pipeline(self, packet_id, raw_signal):
        # 1. MAC 계층 스케줄링
        scheduled_data = self.mac_scheduler(raw_signal)
        # 2. RLC/PDCP 계층 처리 (바이너리 인코딩)
        final_packet = self.rlc_layer_processing(packet_id, scheduled_data)
        return final_packet

# --- 실행부 (EKS Graviton Pod 내부에서 동작) ---
l2l3 = L2L3Stack()
packet_count = 0

print("--- L2/L3 가상 스택 구동 중 (Listening TX Pod...) ---")

try:
    while True:
        # 가상의 TX Pod로부터 받은 IQ 샘플 데이터 (Sionna 결과물)
        # 실제 환경에서는 gRPC나 Shared Memory를 통해 수신합니다.
        mock_iq_samples = np.random.standard_normal(1024).astype(np.complex64)
        
        # Graviton 멀티코어를 활용한 병렬 처리
        future = l2l3.executor.submit(l2l3.process_pipeline, packet_count, mock_iq_samples)
        result = future.result()
        
        if packet_count % 100 == 0:
            print(f"[L2/L3] Packet {packet_count} processed on ARM64 core")
            
        packet_count += 1
        time.sleep(0.01) # 처리 주기 조절
except KeyboardInterrupt:
    print("Stack stopped.")

```
EKS 배치를 위한 핵심 포인트
* 멀티 프로세싱/스레딩: Graviton은 물리적 코어 당 하나의 스레드를 가지므로, ThreadPoolExecutor나 multiprocessing을 사용하여 시뮬레이션 트래픽을 병렬로 처리할 때 성능이 극대화됩니다.
* 데이터 직렬화: TX(x86)에서 L2/L3(ARM)로 데이터를 보낼 때, 엔디안(Endian) 문제가 발생하지 않도록 Protobuf나 Pickle 등을 사용해 직렬화하는 것이 안전합니다.
* 리소스 할당: EKS에서 이 Pod를 띄울 때 반드시 nodeSelector를 통해 arm64 노드로 보내야 합니다.

----

이 구조에서 Graviton(L2/L3)은 직접적인 물리 신호(IQ 샘플)를 건드리기보다, 데이터를 '캡슐화(Encapsulation)'하여 수신측으로 전달하는 데이터 게이트웨이 역할을 합니다.
실제 시스템에서는 크게 두 가지 연결 방식을 사용합니다:
#### 1. 네트워크 통신 방식 (gRPC 또는 UDP) ####
가장 보편적인 방식으로, TX/L2L3/RX가 각각 별도의 Pod로 떠 있을 때 사용합니다.
* 흐름: TX (x86) → gRPC 전송 → L2/L3 (Graviton) → gRPC 전송 → RX (x86)
* Graviton의 역할: TX에서 받은 IQ 데이터를 Protobuf(gRPC) 메시지에 담고, 헤더(Sequence Number 등)를 붙여 RX Pod의 IP 주소로 쏴줍니다.
* 장점: 확장이 쉽고 각 Pod가 서로 다른 노드에 있어도 상관없습니다.

#### 2. 고속 공유 메모리 방식 (Shared Memory + IPC) ####
지연 시간(Latency)을 극단적으로 줄여야 하는 워크샵 시나리오에서 추천합니다.
* 조건: TX, L2L3, RX 파드가 같은 EKS 노드에 배치되어야 합니다.
* 흐름:
    * TX가 공유 메모리(e.g. /dev/shm)에 IQ 샘플을 씁니다.
    * Graviton(L2/L3)은 메모리 주소 포인터를 받아 "이 데이터는 1번 사용자꺼다"라고 태깅(Scheduling)만 합니다.
    * RX가 해당 주소에서 데이터를 읽어 처리합니다.
* 장점: 데이터 복사(Copy)가 없어 CPU 점유율이 낮고 Graviton의 정수 연산 파워를 오직 제어 로직에만 집중할 수 있습니다.
  
💻 Python 전송 샘플 (gRPC 기반)
Graviton Pod에서 처리한 데이터를 RX Pod로 넘겨주는 핵심 로직입니다.
```
import grpc
import signal_pb2 # gRPC 정의 파일
import signal_pb2_grpc

def send_to_rx(processed_data, packet_id):
    # RX Pod의 서비스 엔드포인트로 연결
    with grpc.insecure_channel('rx-service:50051') as channel:
        stub = signal_pb2_grpc.SignalForwarderStub(channel)
        
        # Graviton에서 L3 헤더를 붙인 패킷 생성
        payload = signal_pb2.SignalRequest(
            id=packet_id,
            data=processed_data.tobytes(), # 바이트로 직렬화
            timestamp=time.time()
        )
        
        # RX로 전송
        response = stub.Forward(payload)
        return response.success
```

💡 워크샵 설계 팁
* 실제 NVIDIA Aerial 가속기를 모사한다면, Graviton은 "데이터가 어디로 갈지 이정표를 세워주는 역할"을 하고, 실제 데이터는 NVLink나 RDMA 같은 고속 통로로 흐르게 설계하는 것이 고수의 방식입니다.

----

### Graviton을 거쳐 RX로 가는 현실적인 데이터 흐름 ###
GPU 노드와 Graviton 노드 사이의 통신은 다음 3단계로 이루어집니다.
* Serialization (직렬화): TX(x86)에서 생성된 complex64 데이터를 바이트 스트림으로 변환합니다.
* L2/L3 Processing (Graviton): 네트워크 패킷 형태로 들어온 데이터를 Graviton이 수신하여 프로토콜 처리(헤더 검사, 스케줄링 결정 등)를 수행합니다.
* Forwarding (포워딩): 처리가 끝난 패킷을 RX(x86)의 IP로 쏴줍니다.

🚀 성능 최적화 대안: gRPC 대신 'UDP Raw Socket'
* Sionna 실시간 신호 전송처럼 데이터 양이 많을 때는 TCP 기반의 gRPC보다 UDP가 유리합니다. 특히 Graviton 노드에서 고성능 네트워크 인터페이스(ENA)를 활용하면 패킷 손실을 최소화하며 전송할 수 있습니다.

