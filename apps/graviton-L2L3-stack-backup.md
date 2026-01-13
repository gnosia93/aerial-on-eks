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

* 데이터 직렬화: TX(x86)에서 L2/L3(ARM)로 데이터를 보낼 때, 엔디안(Endian) 문제가 발생하지 않도록 Protobuf나 Pickle 등을 사용해 직렬화하는 것이 안전합니다.
* 이 구조에서 Graviton(L2/L3)은 직접적인 물리 신호(IQ 샘플)를 건드리기보다, 데이터를 '캡슐화(Encapsulation)'하여 수신측으로 전달하는 데이터 게이트웨이 역할을 한다.
실제 시스템에서는 크게 두 가지 연결 방식을 사용한다.

#### 1. 네트워크 통신 방식 (gRPC 또는 UDP) ####
가장 보편적인 방식으로, TX/L2L3/RX가 각각 별도의 Pod로 떠 있을 때 사용합니다.
* 흐름: TX (x86) → gRPC 전송 → L2/L3 (Graviton) → gRPC 전송 → RX (x86)
* Graviton의 역할: TX에서 받은 IQ 데이터를 Protobuf(gRPC) 메시지에 담고, 헤더(Sequence Number 등)를 붙여 RX Pod의 IP 주소로 쏴준다.

  

### Graviton을 거쳐 RX로 가는 현실적인 데이터 흐름 ###
GPU 노드와 Graviton 노드 사이의 통신은 다음 3단계로 이루어집니다.
* Serialization (직렬화): TX(x86)에서 생성된 complex64 데이터를 바이트 스트림으로 변환합니다.
* L2/L3 Processing (Graviton): 네트워크 패킷 형태로 들어온 데이터를 Graviton이 수신하여 프로토콜 처리(헤더 검사, 스케줄링 결정 등)를 수행합니다.
* Forwarding (포워딩): 처리가 끝난 패킷을 RX(x86)의 IP로 쏴줍니다.

🚀 성능 최적화 대안: gRPC 대신 'UDP Raw Socket'
* Sionna 실시간 신호 전송처럼 데이터 양이 많을 때는 TCP 기반의 gRPC보다 UDP가 유리합니다. 특히 Graviton 노드에서 고성능 네트워크 인터페이스(ENA)를 활용하면 패킷 손실을 최소화하며 전송할 수 있습니다.

```
# 1. TX Pod (Sionna - x86 + GPU)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sionna-tx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: tx-node
  template:
    metadata:
      labels:
        app: tx-node
    spec:
      nodeSelector:
        kubernetes.io/arch: amd64 # x86 노드 강제
        nvidia.com: "true"
      containers:
      - name: tx-container
        image: <your-x86-sionna-image>
        env:
        - name: L2L3_ENDPOINT
          value: "l2l3-service:5000" # Graviton 서비스 주소
        resources:
          limits:
            nvidia.com: 1
---
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

#### 2. Pod 간 통신 설계 (UDP/Binary 전송) ###
아키텍처(x86 ↔ ARM)가 다르므로 바이너리 엔디안(Little-Endian)을 맞춘 UDP 전송이 가장 빠릅니다.

[TX 쪽: Sionna 신호 송신]
```
import socket
import numpy as np

# UDP 설정
L2L3_ADDR = ("l2l3-service", 5000)
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)

def send_signal(iq_samples):
    # complex64 데이터를 바이너리로 직렬화 (Little-endian 보장)
    payload = iq_samples.tobytes() 
    sock.sendto(payload, L2L3_ADDR)
```

[Graviton 쪽: L2/L3 데이터 중계]
```
import socket
import numpy as np

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.bind(('0.0.0.0', 5000))

print("[Graviton] L2/L3 Stack Listening...")

while True:
    data, addr = sock.recvfrom(65535) # 최대 MTU 고려
    # 1. 수신 데이터를 numpy로 복원 (ARM에서 처리)
    iq_samples = np.frombuffer(data, dtype=np.complex64)
    
    # 2. L2/L3 제어 로직 (Scheduling 등) 수행
    # ... logic ...
    
    # 3. RX Pod(x86)로 다시 토스
    sock.sendto(data, ("rx-service", 5001))
```
워크샵 성공을 위한 팁
* 이미지 빌드: docker buildx를 사용하여 linux/amd64(TX/RX용)와 linux/arm64(L2L3용) 이미지를 각각 빌드하여 Amazon ECR에 밀어넣어야 합니다.
인스턴스 타입 추천:
* TX/RX: g5.xlarge (NVIDIA A10G)
* L2/L3: c7g.medium 또는 c7gn.medium (네트워크 최적화형 Graviton)
* 네트워크 병목: 데이터 양이 너무 많으면 SOCK_DGRAM 대신 NVIDIA DPDK를 고려해야 하지만, 워크샵 수준에서는 UDP만으로도 충분합니다.
이렇게 구성하면 "AI/신호처리는 GPU(x86)가, 통신 프로토콜은 가성비의 Graviton(ARM)이" 담당하는 최신형 O-RAN 시뮬레이션 구조가 완성됩니다.

----
RX(수신) Pod는 Graviton(L2/L3)으로부터 전달받은 바이너리 패킷을 다시 Sionna/TensorFlow가 이해할 수 있는 복소수 텐서로 복원하고, 최종적으로 신호를 복조(Demodulation)하는 역할을 수행합니다.
x86 환경에서 NVIDIA GPU 가속을 사용하여 데이터를 복원하는 핵심 로직입니다.

```
import socket
import numpy as np
import tensorflow as tf
from sionna.utils import QAMSource, BinarySource
# Sionna의 Mapper/Demapper 설정 (TX와 파라미터 동일해야 함)
from sionna.mapping import Mapper, Demapper

class SignalReceiver(tf.keras.Model):
    def __init__(self, num_bits_per_symbol=4):
        super().__init__()
        self.demapper = Demapper("app", constellation_type="qam", num_bits_per_symbol=num_bits_per_symbol)
        # 비교를 위한 소스 (BER 계산용)
        self.binary_source = BinarySource()

    def decode_signal(self, iq_samples_raw, ebno_db=20.0):
        # 1. 바이너리 데이터를 complex64로 변환
        iq_samples = np.frombuffer(iq_samples_raw, dtype=np.complex64)
        
        # 2. 텐서로 변환 및 GPU로 전송
        y = tf.convert_to_tensor(iq_samples)
        y = tf.reshape(y, [64, -1]) # TX에서 보낸 Batch size와 동일하게 맞춰줌
        
        # 3. 노이즈 추정치 설정 (AWGN 기준)
        no = tf.cast(10**(-ebno_db/10), tf.float32)
        
        # 4. Demapping (IQ 샘플 -> LLR/Bits)
        llr = self.demapper([y, no])
        
        # 5. 최종 비트 결정 (Hard Decision)
        bits_hat = tf.cast(llr < 0, tf.int32)
        return bits_hat

# --- RX Pod 실행부 ---
rx = SignalReceiver()
sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
sock.bind(('0.0.0.0', 5001)) # L2/L3(Graviton)이 쏴주는 포트

print("[RX] Receiver is ready (x86 + GPU)")

while True:
    # 1. Graviton으로부터 데이터 수신
    data, addr = sock.recvfrom(65535) 
    
    # 2. 신호 복원 및 비트 추출
    with tf.device('/GPU:0'): # GPU 가속 강제
        recovered_bits = rx.decode_signal(data)
    
    # 3. 결과 출력 (샘플)
    print(f"[RX] Recovered Shape: {recovered_bits.shape} on GPU")

```
구조적 핵심 포인트
* 데이터 무결성: TX → Graviton → RX를 거치면서 데이터가 깨지지 않도록 np.frombuffer 호출 시 반드시 dtype=np.complex64를 명시해야 합니다. NVIDIA Sionna는 기본적으로 float32 기반의 복소수를 사용합니다.
* GPU 컨텍스트: RX Pod는 수신 대기 상태(recvfrom)에서 CPU를 쓰다가, 데이터가 들어오는 즉시 tf.device('/GPU:0')를 통해 NVIDIA GPU로 연산을 밀어 넣어 처리 지연을 최소화합니다.
* 병목 현상 방지: UDP 패킷 크기가 너무 작으면 잦은 GPU 전송 오버헤드가 발생합니다. Graviton에서 패킷을 어느 정도 모아서 보내거나(Batching), Shared Memory가 불가능한 환경이므로 Jumbo Frame 설정을 EKS 노드에 적용하는 것이 좋습니다.
* 이제 전체 파이프라인(TX-Sionna → L2/L3-Graviton → RX-PyAerial)이 연결되었습니다. 이 환경을 한 번에 배포할 수 있는 Helm Chart 구조를 잡아드릴까요? 아니면 성능 모니터링(BER 측정) 로직을 추가해 볼까요?
----

워크샵의 완성도를 높이기 위해, 멀티 아키텍처 노드 배치(Node Affinity)를 지원하는 Helm Chart와 실시간으로 통신 품질을 측정하는 성능 모니터링(BER/Throughput) 로직을 통합해 드립니다.

#### 1. Helm Chart 구조 (Sionna Multi-Arch) ####
values.yaml에서 아키텍처별 설정을 관리하고, 각 템플릿에서 이를 할당합니다.
[values.yaml]
```
global:
  domain: "sionna.workshop"

tx:
  image: "your-repo/sionna-tx-x86:latest"
  gpuLimit: 1
  ebno_db: 20.0

l2l3: # Graviton 전용
  image: "your-repo/l2l3-stack-arm64:latest"
  replicaCount: 2

rx:
  image: "your-repo/sionna-rx-x86:latest"
  gpuLimit: 1
```

[templates/tx-deployment.yaml (GPU/x86 노드 강제)]
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-tx
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/arch: amd64
        nvidia.com: "true"
      containers:
      - name: tx
        image: {{ .Values.tx.image }}
        resources:
          limits:
            nvidia.com: {{ .Values.tx.gpuLimit }}

```

## 2. 성능 모니터링 로직 (RX 측 BER 측정) ##
수신단(RX)에서 송신 데이터와 대조하여 비트 에러율(BER)을 계산하고 이를 실시간으로 출력하거나 시각화 툴로 보낼 수 있는 코드입니다.

rx_logic.py
```
import tensorflow as tf
import numpy as np
from sionna.utils import BinarySource

class PerformanceMonitor:
    def __init__(self):
        print("[Monitor] Performance Tracker Initialized")

    def calculate_ber(self, original_bits, recovered_bits):
        """
        비트 에러율(Bit Error Rate) 계산
        """
        # 0과 1의 차이를 계산하여 에러 개수 파악
        error_count = tf.reduce_sum(tf.abs(original_bits - recovered_bits))
        total_bits = tf.cast(tf.size(original_bits), tf.float32)
        ber = error_count / total_bits
        return ber.numpy()

# RX 루프 내 적용 예시
monitor = PerformanceMonitor()
source = BinarySource()

while True:
    data, addr = sock.recvfrom(65535)
    
    # 1. 수신 및 복조 (이전 단계 코드 참조)
    recovered_bits = rx.decode_signal(data)
    
    # 2. 성능 평가 (실제 환경선 TX가 보낸 시퀀스 넘버 기반으로 대조)
    # 여기서는 데모를 위해 동일 크기의 랜덤 소스와 비교 예시
    mock_original = source([64, 1024]) 
    current_ber = monitor.calculate_ber(mock_original, recovered_bits)
    
    print(f"📊 [Real-time Stats] BER: {current_ber:.6f} | Status: {'Stable' if current_ber < 0.01 else 'High Error'}")

```

### 전체 아키텍처 워크플로우 요약 ###
* AWS EKS 클러스터에 x86 GPU 노드그룹과 Graviton ARM 노드그룹을 각각 생성합니다.
* Helm을 통해 helm install sionna-workshop ./chart 명령어로 전체 파이프라인을 배포합니다.
* TX Pod가 NVIDIA TensorFlow 환경에서 신호를 생성해 Graviton L2/L3 Pod로 쏩니다.
* Graviton은 저렴한 비용으로 패킷을 중계/스케줄링하고 RX Pod로 전달합니다.
* RX Pod는 다시 GPU 가속을 통해 신호를 복원하고 BER 수치를 모니터링 대시보드(예: Prometheus/Grafana)로 전송합니다.

----

EKS 환경에서 Graviton(L2/L3)과 GPU(TX/RX) 노드 사이의 성능 지표(BER, Throughput)를 실시간으로 수집하기 위해, Prometheus Python Client를 활용한 메트릭 노출 및 ServiceMonitor 설정 방법을 안내해 드립니다.

#### 1. RX Pod에 Prometheus 메트릭 노출 코드 추가 ####
수신단(RX)에서 계산된 BER(비트 에러율)과 처리량(Throughput)을 Prometheus가 긁어갈 수 있도록 엔드포인트를 열어줍니다
```
from prometheus_client import start_http_server, Gauge, Counter
import time

# 1. Prometheus 메트릭 정의
BER_GAUGE = Gauge('sionna_rx_ber', 'Current Bit Error Rate')
THROUGHPUT_COUNTER = Counter('sionna_rx_bits_total', 'Total bits processed')
LATENCY_GAUGE = Gauge('sionna_processing_latency_ms', 'End-to-end latency in ms')

# 2. 메트릭 서버 시작 (8000 포트)
start_http_server(8000)

def monitor_performance(original_bits, recovered_bits, start_time):
    # BER 계산 로직
    error_count = np.sum(np.abs(original_bits - recovered_bits))
    total_bits = recovered_bits.size
    ber = error_count / total_bits
    
    # 3. Prometheus에 값 업데이트
    BER_GAUGE.set(ber)
    THROUGHPUT_COUNTER.inc(total_bits)
    LATENCY_GAUGE.set((time.time() - start_time) * 1000)

```

#### 2. Prometheus ServiceMonitor 설정 (YAML) ####
Prometheus Operator를 사용 중이라면, 아래 설정으로 자동으로 RX Pod의 메트릭을 수집합니다.
```
apiVersion: monitoring.coreos.com
kind: ServiceMonitor
metadata:
  name: sionna-rx-monitor
  labels:
    release: prometheus # 헬름 차트 릴리스 라벨과 일치해야 함
spec:
  selector:
    matchLabels:
      app: rx-node # RX Pod의 라벨
  endpoints:
  - port: metrics # RX Pod에서 8000번 포트에 지정한 이름
    interval: 5s   # 5초마다 실시간 수집

```
3. Grafana 대시보드 구성 팁
* 수집된 데이터를 Grafana에서 시각화할 때 다음 쿼리를 사용하세요:
* 실시간 BER: sionna_rx_ber
* 평균 Throughput (Mbps): rate(sionna_rx_bits_total[1m]) / 1000000
* Graviton-x86 간 지연 시간: sionna_processing_latency_ms

💡 아키텍처 관전 포인트
* Graviton 노드: CloudWatch Agent나 Prometheus를 통해 CPU 사용률을 모니터링하여, L2/L3 로직이 ARM 코어에 얼마나 효율적으로 분산되는지 확인합니다.
* GPU 노드: NVIDIA DCGM Exporter를 함께 띄우면 Sionna 연산 시 GPU 이용률(Utilization)과 메모리 점유율을 동시에 관찰할 수 있습니다.

----
* Grafana Alerting을 사용하면 Sionna 시뮬레이션 중 채널 환경이 악화되거나 코딩 성능이 떨어져 BER(비트 에러율)이 임계치를 넘었을 때 즉시 알림(Slack, Email 등)을 받을 수 있습니다.
---

#### 1. L2/L3가 없어도 되는 경우 (단순 물리 계층 연구) ####
단순히 "이 안테나 알고리즘이 에러를 얼마나 줄이는가?" 또는 "AI 디모듈레이터 성능이 좋은가?"를 보고 싶다면 TX에서 RX로 바로 쏴도 됩니다.
* 구조: TX (Sionna) → Network → RX (Sionna/PyAerial)
* 장점: 구조가 단순하고 지연 시간이 짧습니다. Sionna 공식 튜토리얼의 대부분은 이 방식입니다.

#### 2. 그럼에도 L2/L3를 넣는 이유 (진짜 통신 시스템 모사) ####
워크샵의 주제가 "시스템 엔지니어링"이나 "O-RAN 구조 이해"라면 L2/L3는 다음과 같은 필수 역할을 합니다.
* 스케줄링 (Resource Allocation): TX가 신호를 막 쏘는 게 아니라, "지금 1번 유저는 상태가 좋으니 데이터를 많이 보내고, 2번은 안 좋으니 조금만 보내"라고 결정하는 두뇌가 필요합니다.
* 패킷 재전송 (HARQ): RX에서 에러가 났을 때 "다시 보내줘!"라고 TX에 요청하고, TX가 그걸 기억했다가 다시 보내주는 로직을 처리합니다.
* 상위 레이어 연결: 실제 인터넷 데이터(YouTube 영상 등)를 가져와서 무선 신호에 실으려면 IP 패킷을 무선 프레임으로 변환하는 계층이 반드시 필요합니다.

요약 및 제안
* 단순 알고리즘 검증이 목표라면? → L2/L3를 과감히 빼고 TX-RX를 직접 연결하세요.
* 전체 기지국 아키텍처(O-RAN) 시뮬레이션이 목표라면? → L2/L3를 유지하세요.

O-RAN(Open Radio Access Network, 개방형 무선 접속망)은 기지국을 구성하는 하드웨어와 소프트웨어를 분리하고, 그 사이의 인터페이스를 표준화하여 서로 다른 제조사의 장비가 연동될 수 있게 하는 기술입니다. O-RAN Alliance가 이 표준화를 주도하고 있습니다
