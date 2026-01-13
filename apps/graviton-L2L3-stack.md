## 그라비톤 기반 L2/L3 스택 구현 ##

L2/L3 스택은 수학적 행렬 연산보다는 패킷의 순서를 맞추고(RLC), 사용자별로 자원을 할당(MAC Scheduler)하며, IP 패킷을 캡슐화하는 복잡한 논리 처리가 주를 이룬다. 아래는 Graviton(ARM64) 노드에서 실행하기 적합한 파이썬 기반의 단순화된 L2(MAC) 스케줄러 및 데이터 처리 샘플 코드로 TX Pod로부터 받은 데이터를 처리하여 RX Pod로 전달하는 "중계 및 제어" 역할을 수행한다.
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

l2l3 = L2L3Stack()
packet_count = 0

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

* 데이터 직렬화: TX(x86)에서 L2/L3(ARM)로 데이터를 보낼 때, 엔디안(Endian) 문제가 발생하지 않도록 Protobuf나 Pickle 등을 사용해 직렬화하는 것이 안전하다.
* 이 구조에서 Graviton(L2/L3)은 직접적인 물리 신호(IQ 샘플)를 건드리기보다, 데이터를 '캡슐화(Encapsulation)'하여 수신측으로 전달하는 데이터 게이트웨이 역할을 한다.

```
# 2. L2/L3 스택 Pod (Graviton - ARM64)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: l2l3-stack
spec:
  replicas: 1
  selector:
    matchLabels:
      app: l2l3-node
  template:
    metadata:
      labels:
        app: l2l3-node
    spec:
      nodeSelector:
        kubernetes.io/arch: arm64 # Graviton 노드 강제
      containers:
      - name: l2l3-container
        image: <your-arm64-l2l3-image>
        ports:
        - containerPort: 5000 # TX로부터 데이터 받는 포트
---
# 3. L2/L3 서비스 (내부 통신용)
apiVersion: v1
kind: Service
metadata:
  name: l2l3-service
spec:
  selector:
    app: l2l3-node
  ports:
  - protocol: UDP
    port: 5000
    targetPort: 5000
```

## 기타 ##
### Graviton을 거쳐 RX로 가는 현실적인 데이터 흐름 ###
GPU 노드와 Graviton 노드 사이의 통신은 다음 3단계로 이루어집니다.
* Serialization (직렬화): TX(x86)에서 생성된 complex64 데이터를 바이트 스트림으로 변환합니다.
* L2/L3 Processing (Graviton): 네트워크 패킷 형태로 들어온 데이터를 Graviton이 수신하여 프로토콜 처리(헤더 검사, 스케줄링 결정 등)를 수행합니다.
* Forwarding (포워딩): 처리가 끝난 패킷을 RX(x86)의 IP로 쏴줍니다.

### 네트워크 통신 방식 (gRPC 또는 UDP) ###
가장 보편적인 방식으로, TX/L2L3/RX가 각각 별도의 Pod로 떠 있을 때 사용합니다.
* 흐름: TX (x86) → gRPC 전송 → L2/L3 (Graviton) → gRPC 전송 → RX (x86)
* Graviton의 역할: TX에서 받은 IQ 데이터를 Protobuf(gRPC) 메시지에 담고, 헤더(Sequence Number 등)를 붙여 RX Pod의 IP 주소로 쏴준다.

🚀 성능 최적화 대안: gRPC 대신 'UDP Raw Socket'
* Sionna 실시간 신호 전송처럼 데이터 양이 많을 때는 TCP 기반의 gRPC보다 UDP가 유리합니다. 특히 Graviton 노드에서 고성능 네트워크 인터페이스(ENA)를 활용하면 패킷 손실을 최소화하며 전송할 수 있습니다.


