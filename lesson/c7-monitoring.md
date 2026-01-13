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
