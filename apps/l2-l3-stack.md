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


