## 모니터링 (RX BER 측정) ##
수신단(RX)에서 송신 데이터와 대조하여 비트 에러율(BER)을 계산하고 이를 실시간으로 출력하거나 시각화 툴로 보낼 수 있는 코드이다

#### 참조 데이터 (Reference Data) 확보 ####
BER을 측정하려면, 수신하여 디코딩한 비트 스트림(decoded_bits)과 원래 송신측에서 보냈던 원본 비트 스트림을 비교해야 하는데. 이 원본 데이터를 참조 데이터(Reference Data)라고 한다.
* 실제 시스템: 실제 통신에서는 미리 약속된 특정 패턴(테스트 PRBS 패턴)이나, 상위 계층에서 확인된 정상적인 데이터 블록을 참조 데이터로 사용
* 시뮬레이션/테스트 환경: TX에서 보낸 데이터를 RX가 접근할 수 있는 공유 메모리나 네트워크 경로로 미리 전달

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

### RX Pod에 Prometheus 메트릭 노출 코드 추가 ###
수신단(RX)에서 계산된 BER(비트 에러율)과 처리량(Throughput)을 Prometheus가 긁어갈 수 있도록 엔드포인트를 열어 준다.
```
from prometheus_client import start_http_server, Gauge, Counter
import time

# 1. Prometheus 메트릭 정의
BER_GAUGE = Gauge('sionna_rx_ber', 'Current Bit Error Rate')
THROUGHPUT_COUNTER = Counter('sionna_rx_bits_total', 'Total bits processed')
LATENCY_GAUGE = Gauge('sionna_processing_latency_ms', 'End-to-end latency in ms')

# 메트릭 서버 시작 (8000 포트)
start_http_server(8000)

def monitor_performance(original_bits, recovered_bits, start_time):
    # BER 계산 로직
    error_count = np.sum(np.abs(original_bits - recovered_bits))
    total_bits = recovered_bits.size
    ber = error_count / total_bits
    
    # Prometheus에 값 업데이트
    BER_GAUGE.set(ber)
    THROUGHPUT_COUNTER.inc(total_bits)
    LATENCY_GAUGE.set((time.time() - start_time) * 1000)
```

### Prometheus ServiceMonitor 설정 ###
Prometheus Operator를 사용 중이라면, 아래 설정으로 자동으로 RX Pod의 메트릭을 수집한다.
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


### Grafana 대시보드 구성 ###
* 실시간 BER: sionna_rx_ber
* 평균 Throughput (Mbps): rate(sionna_rx_bits_total[1m]) / 1000000
* Graviton-x86 간 지연 시간: sionna_processing_latency_ms


